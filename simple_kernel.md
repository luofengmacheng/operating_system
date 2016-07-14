## 操作系统是如何工作的

[mykernel](https://github.com/mengning/mykernel)

### 1 课程内容

本期主要讲了两个内容：

* 函数调用堆栈
* 中断

关于函数调用堆栈，上期内容中已经讲到了，本期则通过几个代码例子重点进行了讲解，下面进行下梳理：

* 函数调用堆栈是从高地址向低地址进行扩展，ebp指向当前调用函数的`基地址`，esp指向当前调用函数的`栈顶地址`
* 参数的传递方式见第2节
* 函数的返回值存放到eax寄存器
* 当调用一个函数时需要进行以下几个步骤：参数入栈；保存返回地址，并将eip置为调用函数的地址；保存栈基。同样，当函数返回时：清空栈；将eip设置为保存的返回地址。

### 2 函数的调用规则

函数的调用规则指示了参数传递的顺序，以及该由谁把参数出栈。

* __cdecl：参数`从右到左`压栈，由`调用者`把参数出栈，返回值在eax中。
* __stdcall：参数`从右到左`压栈，由`被调用者`把参数出栈，返回值在eax中。
* __fastcall：前两个参数是`从左到右`通过寄存器传递的，分别是ecx和edx，其它的参数是`从右到左`压栈，由`被调用者`把参数出栈，返回值在eax中。
* __pascal：参数`从左向右`传递参数，由`调用者`把参数出栈，返回值在eax中。
* __thiscall：用于C++成员函数，this指针存放于eax寄存器，参数`由右到左`压栈，但是，该约定不能被程序员使用。

Linux下默认是__cdecl，通过在函数名前添加__attribute__((__stdcall))可以修改调用规则。

### 3 关于中断

为什么需要中断？中断能够带来什么好处？

早起的程序设计称为`单道程序设计`，只能等待前一个程序执行完成后才能执行后一个程序，那么，如果前一个程序执行的时间很长，就要等待很久才能执行后一个程序。

中断能够解决单道程序的问题，当一个程序正在运行时，可以通过中断，执行中断处理程序，保存现场，再执行新的程序。这样就能够使得多个程序`并行运行`，从而使得`多道程序设计`成为可能。

### 4 Linux内核基础上实现时间片轮转进程调度

首先，解释一下时间片轮转调度：每个进程运行一个时间片，如果一个时间片结束时，进程还在运行，就强行剥夺进程的执行权，调度给下一个就绪态进程，如果时间片还没到，进程已经结束了，则立刻调度下一个就绪态进程。

时间片轮转的关键问题是：时间片的长度。时间片过短，会导致过多的进程切换，而进程切换是需要开销的，降低了利用率；时间片过长，会使得有些进程需要更长的时间才能获得响应。

想象一下计算机的运行，当系统启动后，有一个进程一直在运行，而且它是系统所有进程的祖先进程，它就是init进程，当发生时钟中断时，系统检测一下是否有其它的就绪态，如果有则调度下一个就绪态进程。

因此，大体上，这里必须实现两个程序：

* init进程
* 时钟中断处理程序

下面通过分析补丁的代码进行讲解。

实现时间片轮转进程调度需要修改6个文件：

* include/linux/timer.h
* arch/x86/kernel/time.c
* include/linux/start_kernel.h
* init/main.c
* Makefile
* mykernel/mymain.c
* mykernel/myinterrupt.c

其中，前面四个是linux内核自身的文件，需要修改内核以便让内核认识我们要添加的函数，以便在适当的时机调用，中间的Makefile是让linux在编译时编译我们自己的代码，后面两个是我们自己的代码。

#### 4.1 timer.h

在里面声明我们的时钟中断处理程序my_timer_handler()。

#### 4.2 time.c

time.c中timer_interrupt是默认的时钟中断处理程序，因此，要想让编写一个时钟

在timer_interrupt()函数中调用我们的时钟中断处理程序my_timer_handler()。

#### 4.3 start_kernel.h

在里面声明我们的启动调用函数my_start_kernel()。

#### 4.4 main.c

在start_kernel()函数的最后调用my_start_kernel()。

#### 4.5 Makefile

将mykernel目录添加到编译目录。

#### 4.7 mymain.c

``` C
/* mypcb.h */
#define MAX_TASK_NUM        4
#define KERNEL_STACK_SIZE   1024*8

struct Thread {
    unsigned long		ip;  /* EIP */
    unsigned long		sp;  /* ESP */
};

/* 进程控制块 */
typedef struct PCB{
    int pid;                          /* 进程ID */
    volatile long state;	          /* 进程状态：-1 阻塞, 0 运行态, >0 就绪态 */
    char stack[KERNEL_STACK_SIZE];    /* 进程的堆栈 */
    /* CPU-specific state of this task */
    struct Thread thread;             /* 保存进程的EIP和ESP */
    unsigned long	task_entry;       /* 进程入口函数 */
    struct PCB *next;                 /* 下一个进程控制块 */
}tPCB;

void my_schedule(void);
```

``` C
/* mymain.c */
#include "mypcb.h"

/* task保存所有进程的信息 */
tPCB task[MAX_TASK_NUM];

/* 当前运行的进程 */
tPCB * my_current_task = NULL;

/* 是否需要调度标志 */
volatile int my_need_sched = 0;

void my_process(void);

/* 系统启动时调用的函数 */
void __init my_start_kernel(void)
{
    int pid = 0;
    int i;
    /* 初始化进程控制块链表 */
    task[pid].pid = pid;
    task[pid].state = 0;/* -1 unrunnable, 0 runnable, >0 stopped */
    /* 开始时，进程的入口和EIP都是my_process */
    task[pid].task_entry = task[pid].thread.ip = (unsigned long)my_process;
    task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
    /* 开始只有一个进程，它的下一个进程还是它本身 */
    task[pid].next = &task[pid];
    /* 创建MAX_TASK_NUM个进程 */
    for(i=1;i<MAX_TASK_NUM;i++)
    {
        memcpy(&task[i],&task[0],sizeof(tPCB));
        task[i].pid = i;
        task[i].state = -1;
        task[i].thread.sp = (unsigned long)&task[i].stack[KERNEL_STACK_SIZE-1];
        task[i].next = task[i-1].next;
        task[i-1].next = &task[i];
    }
    /* 启动ID为0的进程(init) */
    pid = 0;
    my_current_task = &task[pid];

    /* 建立init进程的环境：将进程的栈顶地址放到esp和ebp，将eip设置为当前的指令地址 */
	asm volatile(
    	"movl %1,%%esp\n\t" 	/* 将进程的栈顶地址给esp */
    	"pushl %1\n\t" 	        /* 将进程的栈顶地址压栈 */
    	"pushl %0\n\t" 	        /* 将进程的指令地址压栈 */
    	"ret\n\t" 	            /* 出栈，并将内容给eip，实际是将进程的指令地址放到eip */
    	"popl %%ebp\n\t"        /* 出栈，并将内容给ebp，实际是将进程的栈顶地址给ebp，因为，此时的栈顶地址等于ebp */
    	: 
    	: "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)	/* 将进程的指令地址放到ecx，进程的堆栈地址放到edx*/
	);
}

/* init进程的入口函数 */
void my_process(void)
{
    int i = 0;
    while(1)
    {
        i++;
        if(i%10000000 == 0)
        {
            printk(KERN_NOTICE "this is process %d -\n",my_current_task->pid);
            /* 查看是否需要调度 */
            if(my_need_sched == 1)
            {
            	/* 重置调度标志，进行调度 */
                my_need_sched = 0;
        	    my_schedule();
        	}
        	printk(KERN_NOTICE "this is process %d +\n",my_current_task->pid);
        }     
    }
}
```

#### 4.6 myinterrupt.c

``` C
/* myinterrupt.c */
#include "mypcb.h"

extern tPCB task[MAX_TASK_NUM];
extern tPCB * my_current_task;
extern volatile int my_need_sched;
volatile int time_count = 0;

/* 时钟中断处理程序 */
void my_timer_handler(void)
{
#if 1
    if(time_count%1000 == 0 && my_need_sched != 1)
    {
    	/* 中断100次才进行调度 */
        printk(KERN_NOTICE ">>>my_timer_handler here<<<\n");
        my_need_sched = 1;
    } 
    time_count ++ ;  
#endif
    return;  	
}

/* 进程调度程序 */
void my_schedule(void)
{
    tPCB * next;
    tPCB * prev;

    if(my_current_task == NULL 
        || my_current_task->next == NULL)
    {
    	return;
    }
    printk(KERN_NOTICE ">>>my_schedule<<<\n");
    /* prev是当前进程，next是下一个进程 */
    next = my_current_task->next;
    prev = my_current_task;
    if(next->state == 0)/* -1 unrunnable, 0 runnable, >0 stopped */
    {
    	/* 如果进程状态是就绪态，则执行，将当前进程修改为下一个进程 */
    	my_current_task = next;
    	printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);
    	/* 进程切换 */
    	asm volatile(
        	"pushl %%ebp\n\t" 	    /* 将当前进程的栈基地址压栈 */
        	"movl %%esp,%0\n\t" 	/* 将当前进程的esp写入到当前进程的sp中，即保存当前进程的esp */
        	"movl %2,%%esp\n\t"     /* 将下一个要执行的进程的sp载入到esp */
        	"movl $1f,%1\n\t"       /* 将标号1:的地址放到ip */
        	"pushl %3\n\t"          /* 将下一个要执行的进程的ip压栈 */
        	"ret\n\t" 	            /* 将下一个要执行的进程的ip载入到eip */
        	"1:\t"                  /* next process start here */
        	"popl %%ebp\n\t"
        	: "=m" (prev->thread.sp),"=m" (prev->thread.ip)
        	: "m" (next->thread.sp),"m" (next->thread.ip)
    	);
 	
    }
    else
    {
    	/* 如果进程状态不是0，说明进程还没有启动，则启动一个新的进程 */
        next->state = 0;
        my_current_task = next;
        printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);
    	/* switch to new process */
    	asm volatile(	
        	"pushl %%ebp\n\t" 	    /* 将当前进程的栈基地址压栈 */
        	"movl %%esp,%0\n\t" 	/* 保存当前进程的栈顶地址esp */
        	"movl %2,%%esp\n\t"     /* 将下一个要执行的进程的sp载入esp */
        	"movl %2,%%ebp\n\t"     /* 将下一个要执行的进程的sp载入ebp */
        	"movl $1f,%1\n\t"       /* 保存当前进程的eip */
        	"pushl %3\n\t"
        	"ret\n\t" 	            /* 与上一条push命令一起实现：将下一个要执行的进程的ip放入到eip */
        	: "=m" (prev->thread.sp),"=m" (prev->thread.ip)
        	: "m" (next->thread.sp),"m" (next->thread.ip)
    	);
    }
    return;
}
```

### 5 关于以上部分内联汇编的解释

内联汇编的基本格式：

```
asm (
	函数体
	: 输出
	: 输入
	: 破坏部分
)
```

* %0，输出和输入中排在第0个位置，对输出和输入进行排序，依次是0、1、2等等。
* "=m"，其中"="表示输出，"m"表示内存(这个结果放到内存中)。

### 6 对以上内联汇编的部分问题

* 问题1：movl $1f,%1\n\t，其中的$1是什么意思？

$1f是标号为1:后面一行的代码的地址。

* 问题2：my_schedule()函数中，当当前进程的状态为0时，ret后面的语句仍然可以执行？

当要进行进程切换时，先将当前进程的ebp压栈，接着，保存esp，并载入下一个进程的esp，将popl %%ebp\n\t的地址保存到当前进程的ip，然后载入下一个进程的eip。那么，当下一次进程切换时，就会首先执行popl %%ebp\n\t，建立ebp。因此，movl $1f,%1\n\t的含义就是保存返回地址，在下一次进程切换时，会切换到该条语句执行，重建ebp。

* 问题3：my_schedule()函数中，当当前进程的状态为1时，也有`movl $1f,%1\n\t`，但是，为什么没有"1:\t"？

表示将汇编语句的最后作为返回地址。

### 7 实验过程中遇到的问题

个人在虚拟机上搭建环境，遇到了几个问题：

* 执行gcc -o init linktable.c menu.c test.c -m32 -static –lpthread语句时，报：

```
-lpthread: No such file or directory
```

一直以为是缺少pthread库，但是，发现系统中有该库，后来在网上查找，说是缺少glibc，但是，我也已经安装了glibc，最后测试了下，是缺少glibc-static库。

```
yum install -y glibc-static.i686
```

* 启动虚拟机(qemu -kernel linux-3.18.6/arch/x86/boot/bzImage -initrd rootfs.img)时，报:

```
VNC server running on '::1;5900'
```

就是出不来虚拟机的窗口。

解决：安装tigervnc(yum install -y tigervnc.x86_64)，然后在可视化界面执行

```
vncviewer localhost::5900
```