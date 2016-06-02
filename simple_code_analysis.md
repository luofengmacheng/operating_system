## 一段简单的C程序分析

### 1 汇编的基本知识

常用的寄存器(32位)：

* eax: 累加寄存器，并且作为函数的返回值。
* eip: 指令寄存器，存储下一条指令的地址。
* ebp: 堆栈基址寄存器，保存当前函数的栈的基地址。
* esp: 堆栈栈顶寄存器，保存当前函数的栈的栈顶地址。

寻址方式：

* 立即数寻址：$10，表示常量10。
* 寄存器寻址：%eax，表示寄存器的值。
* 直接寻址：0x123，访问0x123为地址的值。
* 间接寻址：(%ebp)，表示以ebp寄存器保存的值为地址的值。
* 变址寻址：4(%ebp)，将ebp寄存器中的值加4作为地址的值。

另外需要注意的是：

* 指令通常与英文含义对应，但是，指令的后面会添加位数的标记：movl(32位)
* eip是指令寄存器，但是，不允许程序员直接修改，而是需要通过call、ret等伪指令进行操作。
* 栈地址值是从上到下减小的，也就是说，上面的地址值大，下面的地址值小。
* 栈帧：ebp又称为帧寄存器，esp称为栈寄存器，它们之间的区域称为栈帧，栈帧记录的是当前执行函数的活动。
* 汇编有两种标准：AT&T和intel，最大的不同就是操作数的方向，例如，movl %esp, %ebp，AT&T中是将esp的值给ebp，但是，intel中是将ebp的值给esp。

### 2 C语言编译为汇编

``` C
int g(int x) {
	return x + 3;
}

int f(int x) {
	return g(x);
}

int main() {
	return f(8) + 1;
}
```

通过下面的命令将上面的代码编译为汇编，删除其中的以"点"开头的指令。

```
gcc -S -o main.s main.c -m32
```

汇编代码：

```
g:
    pushl   %ebp
    movl    %esp, %ebp
    movl    8(%ebp), %eax
    addl    $3, %eax
    popl    %ebp
    ret
f:
    pushl   %ebp
    movl    %esp, %ebp
    subl    $4, %esp
    movl    8(%ebp), %eax
    movl    %eax, (%esp)
    call    g
    leave
    ret
main:
    pushl   %ebp
    movl    %esp, %ebp
    pushl   %ecx
    subl    $4, %esp
    movl    $8, (%esp)
    call    f
    addl    $1, %eax
    leave
    ret
```

### 3 对汇编代码的解读

各位看官，我将用图示的方式解析上面汇编代码的执行：

需要注意的是，当CPU正在执行某条指令时，eip指向的应该是下一条指令，但是，这里为了说明当前指令的执行结果，而将eip设置为当前执行的指令。

#### 3.1 开启一个新的栈帧

可以看到，这几个图的开始两条命令都是：

```
pushl %ebp
movl %esp, %ebp
```

将%ebp压栈，再将栈顶地址赋给帧寄存器。表达的含义就是：`开始一个新的栈帧`。

也就是说，当调用一个函数时，需要开始一个新的栈帧，该函数的任何行为都限定在该栈帧中。

![](https://github.com/luofengmacheng/operating_system/blob/master/pics/pic1.png)

#### 3.2 给函数传递参数

函数的参数是`从右到左`依次压入栈中，所以在图中看就是，第一个参数在最下面(8(%ebp))，第二个参数在第一个参数的上面(12(%ebp))。

![](https://github.com/luofengmacheng/operating_system/blob/master/pics/pic2.png)

#### 3.3 函数的调用和返回

函数的调用通过call指令，返回通过ret指令。

call f的含义如下指令：

```
pushl %eip
movl f,%eip
```

将eip的值压栈，再将要调用的函数的地址载入到eip中。

当然，ret指令的含义与call相反：

```
popl %eip
```

出栈，并将出栈的值给eip。

另外还有两个与函数调用相关的指令是enter和leave。

其实，enter指令的含义就是2.1节中提到的两条指令。

enter：

```
pushl %ebp
movl %esp, %ebp
```

保存调用函数的基址，并开始一个新的堆栈。

leave：

```
movl %ebp, %esp
popl %ebp
```

清空当前堆栈，弹出调用函数的基址，并赋给ebp。

![](https://github.com/luofengmacheng/operating_system/blob/master/pics/pic3.png)

![](https://github.com/luofengmacheng/operating_system/blob/master/pics/pic4.png)

#### 2.4 对函数调用的总结

![](https://github.com/luofengmacheng/operating_system/blob/master/pics/pic5.png)

调用函数时要进行三步操作：

* 参数压栈：将参数从右到左压栈。
* call：将指令寄存器的值(下一条指令的地址)压栈，并将指令寄存器的值置为下一条指令的地址。
* enter: 将基址寄存器的值(调用函数的基址)压栈，并开始一个新的栈帧。

同样，函数的返回要进行两步操作：

* leave：将当前栈帧清空，然后，将出栈返回的值赋给基址寄存器。
* ret：再次出栈，并将出栈的值赋给指令寄存器。

但是，也许有的人会发现，f函数中既有leave，又有ret，而g函数中只有ret。

这是因为：根据上面所说的，leave分为两个步骤，一是清空栈，二是将出栈的值赋给基址寄存器，由于g函数中没有调用进行任何变量的声明，也就说它的栈是空的，因此，不需要清空栈，只需要出栈即可，所以g函数中ret前面多了一个popl操作。