## 一段简单的C程序分析

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
    leal    4(%esp), %ecx
    andl    $-16, %esp
    pushl   -4(%ecx)
    pushl   %ebp
    movl    %esp, %ebp
    pushl   %ecx
    subl    $4, %esp
    movl    $8, (%esp)
    call    f
    addl    $1, %eax
    addl    $4, %esp
    popl    %ecx
    popl    %ebp
    leal    -4(%ecx), %esp
    ret
```

![](https://github.com/luofengmacheng/operating_system/blob/master/pics/pic1.png)

![](https://github.com/luofengmacheng/operating_system/blob/master/pics/pic2.png)

![](https://github.com/luofengmacheng/operating_system/blob/master/pics/pic3.png)

![](https://github.com/luofengmacheng/operating_system/blob/master/pics/pic4.png)