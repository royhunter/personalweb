title: Trampolines for Nested Functions in GCC
date: 2016-08-28 21:26:32
tags: Tools
categories: Programme
---
C语言中关于内嵌函数的一些有意思的东西。
<!--more-->

## 遇到的问题
在工作中遇到一段代码crash了，代码大体如下（省去和公司有关的信息）：
```c
int c(int(*d)(int), int z)
{
    d();
    ......
}
int a(int x)
{
    int b(int y)
    {
        return x + y;
    }
    
    c(b, other_arg);
}
```
大意大概是a函数中定义了b函数，b函数是内嵌定义的，而a调用c函数的时候将b函数作为参数传入，直观上看b函数地址是a函数的栈地址空间，c中调用b即是跳转至a的栈中进行执行；而程序却在此部位发生指令异常错误，由于我们用的是mips体系架构的vendor spec gcc，所以第一直觉是认为该mips-gcc并没有正确的对改种函数写法正确的编译；

查阅了相关gcc的手册，确实有对这种内嵌函数的编译进行具体说明的。


## Trampolines for Nested Functions
在gcc手册中针对这种Nested Functions使用了Trampoline技术：
[https://gcc.gnu.org/onlinedocs/gccint/Trampolines.html](https://gcc.gnu.org/onlinedocs/gccint/Trampolines.html)

> A trampoline is **a small piece of code** that is **created at run time** when the address of a nested function is taken. It normally **resides on the stack**, in the stack frame of the containing function. These macros tell GCC how to generate code to allocate and initialize a trampoline.
The instructions in the trampoline must **do two things: load a constant address into the static chain register, and jump to the real address of the nested function**. On CISC machines such as the m68k, this requires two instructions, a move immediate and a jump. Then the two addresses exist in the trampoline as word-long immediate operands. On RISC machines, it is often necessary to load each address into a register in two parts. Then pieces of each address form separate immediate operands.

具体的含义我们在x86的系统中进行了一些实验来进行验证。

## 例1
```c
test1.c
#include <stdio.h>

int a(int x)
{
    int b(int y)
    {
        return x + y;
    }

    return b(x);
}

int main()
{
    printf("%d\n", a(10));
}
```

```bash
$gcc -o test1 test1.c
$objdump -d test1
```

```c
000000000040052d <b.2180>:
  40052d:       55                      push   %rbp
  40052e:       48 89 e5                mov    %rsp,%rbp
  400531:       89 7d fc                mov    %edi,-0x4(%rbp)
  400534:       4c 89 d0                mov    %r10,%rax
  400537:       8b 10                   mov    (%rax),%edx
  400539:       8b 45 fc                mov    -0x4(%rbp),%eax
  40053c:       01 d0                   add    %edx,%eax
  40053e:       5d                      pop    %rbp
  40053f:       c3                      retq   

0000000000400540 <a>:
  400540:       55                      push   %rbp
  400541:       48 89 e5                mov    %rsp,%rbp
  400544:       48 83 ec 10             sub    $0x10,%rsp
  400548:       89 f8                   mov    %edi,%eax
  40054a:       89 45 f0                mov    %eax,-0x10(%rbp)
  40054d:       8b 45 f0                mov    -0x10(%rbp),%eax
  400550:       48 8d 55 f0             lea    -0x10(%rbp),%rdx
  400554:       49 89 d2                mov    %rdx,%r10
  400557:       89 c7                   mov    %eax,%edi
  400559:       e8 cf ff ff ff          callq  40052d <b.2180>   <<<<<<<<<<<调用内嵌函数
  40055e:       c9                      leaveq 
  40055f:       c3                      retq   

0000000000400560 <main>:
  400560:       55                      push   %rbp
  400561:       48 89 e5                mov    %rsp,%rbp
  400564:       bf 0a 00 00 00          mov    $0xa,%edi
  400569:       e8 d2 ff ff ff          callq  400540 <a>
  40056e:       89 c6                   mov    %eax,%esi
  400570:       bf 14 06 40 00          mov    $0x400614,%edi
  400575:       b8 00 00 00 00          mov    $0x0,%eax
  40057a:       e8 91 fe ff ff          callq  400410 <printf@plt>
  40057f:       5d                      pop    %rbp
  400580:       c3                      retq   
  400581:       66 2e 0f 1f 84 00 00    nopw   %cs:0x0(%rax,%rax,1)
  400588:       00 00 00 
  40058b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)
```
在这个例子中可以看到编译出来的代码运行时中规中矩，a调用b的时候只是常规的跳转，只是b的函数名变化了一下，和普通函数并没有什么不同，而且b的函数体也是在代码段中的，与a的栈空间没有任何瓜葛。

## 例2
在例子1的基础上做一些修改：
```c
test2.c
#include <stdio.h>

int aa(int x)
{
    int bb(int y)
    {
        return x + y;
    }

    int (*c)(int z) = bb;    //将内嵌函数进行赋值，同时打印内嵌函数的地址
    printf("c's addr is %p\n", c);

    return c(x);
}


int main()
{
    printf("%d\n", aa(10));
}
```
运行结果：
```bash
c's addr is 0x7fff049e8d94
20
```
从下面test2的pmap空间可以看到，c的地址落于栈中，且栈也具有可执行属性。
```c
3949:   ./test2
0000000000400000      4K r-x-- x1
0000000000600000      4K r-x-- x1
0000000000601000      4K rwx-- x1
00007ffff7a14000   1776K r-x-- libc-2.19.so
00007ffff7bd0000   2044K ----- libc-2.19.so
00007ffff7dcf000     16K r-x-- libc-2.19.so
00007ffff7dd3000      8K rwx-- libc-2.19.so
00007ffff7dd5000     20K rwx--   [ anon ]
00007ffff7dda000    140K r-x-- ld-2.19.so
00007ffff7fe0000     12K rwx--   [ anon ]
00007ffff7ff7000     12K rwx--   [ anon ]
00007ffff7ffa000      8K r-x--   [ anon ]
00007ffff7ffc000      4K r-x-- ld-2.19.so
00007ffff7ffd000      4K rwx-- ld-2.19.so
00007ffff7ffe000      4K rwx--   [ anon ]
00007ffffffde000    132K rwx--   [ stack ]     <<<<<<<<<<<<<<<<<<<<<<<<
ffffffffff600000      4K r-x--   [ anon ]
 total             4196K
```
而这种写法下面的内嵌汇编也复杂了一些：
```c
00000000004005ed <bb.2180>:
  4005ed:       55                      push   %rbp
  4005ee:       48 89 e5                mov    %rsp,%rbp
  4005f1:       89 7d fc                mov    %edi,-0x4(%rbp)
  4005f4:       4c 89 d0                mov    %r10,%rax
  4005f7:       8b 10                   mov    (%rax),%edx
  4005f9:       8b 45 fc                mov    -0x4(%rbp),%eax
  4005fc:       01 d0                   add    %edx,%eax
  4005fe:       5d                      pop    %rbp
  4005ff:       c3                      retq   

0000000000400600 <aa>:
  400600:       55                      push   %rbp
  400601:       48 89 e5                mov    %rsp,%rbp
  400604:       53                      push   %rbx
  400605:       48 83 ec 48             sub    $0x48,%rsp
  400609:       89 f8                   mov    %edi,%eax
  40060b:       64 48 8b 1c 25 28 00    mov    %fs:0x28,%rbx
  400612:       00 00 
  400614:       48 89 5d e8             mov    %rbx,-0x18(%rbp)
  400618:       31 db                   xor    %ebx,%ebx
  40061a:       89 45 c0                mov    %eax,-0x40(%rbp)
  40061d:       48 8d 45 c0             lea    -0x40(%rbp),%rax
  400621:       48 83 c0 04             add    $0x4,%rax
  400625:       48 8d 55 c0             lea    -0x40(%rbp),%rdx
  400629:       b9 ed 05 40 00          mov    $0x4005ed,%ecx
  40062e:       66 c7 00 41 bb          movw   $0xbb41,(%rax)
  400633:       89 48 02                mov    %ecx,0x2(%rax)
  400636:       66 c7 40 06 49 ba       movw   $0xba49,0x6(%rax)         <<<<<<<<<<<<<<<
  40063c:       48 89 50 08             mov    %rdx,0x8(%rax)            <<<<<<<<<<<<<<<
  400640:       c7 40 10 49 ff e3 90    movl   $0x90e3ff49,0x10(%rax)    <<<<<<<<<<<<<<<
  400647:       48 8d 45 c0             lea    -0x40(%rbp),%rax
  40064b:       48 83 c0 04             add    $0x4,%rax
  40064f:       48 89 45 b8             mov    %rax,-0x48(%rbp)
  400653:       48 8b 45 b8             mov    -0x48(%rbp),%rax
  400657:       48 89 c6                mov    %rax,%rsi
  40065a:       bf 44 07 40 00          mov    $0x400744,%edi
  40065f:       b8 00 00 00 00          mov    $0x0,%eax
  400664:       e8 57 fe ff ff          callq  4004c0 <printf@plt>
  400669:       8b 55 c0                mov    -0x40(%rbp),%edx
  40066c:       48 8b 45 b8             mov    -0x48(%rbp),%rax
  400670:       89 d7                   mov    %edx,%edi
  400672:       ff d0                   callq  *%rax                   <<<<<<<<<<<<<<<
  400674:       48 8b 75 e8             mov    -0x18(%rbp),%rsi
  400678:       64 48 33 34 25 28 00    xor    %fs:0x28,%rsi
  40067f:       00 00 
  400681:       74 05                   je     400688 <aa+0x88>
  400683:       e8 28 fe ff ff          callq  4004b0 <__stack_chk_fail@plt>
  400688:       48 83 c4 48             add    $0x48,%rsp
  40068c:       5b                      pop    %rbx
  40068d:       5d                      pop    %rbp
  40068e:       c3                      retq   

000000000040068f <main>:
  40068f:       55                      push   %rbp
  400690:       48 89 e5                mov    %rsp,%rbp
  400693:       bf 0a 00 00 00          mov    $0xa,%edi
  400698:       e8 63 ff ff ff          callq  400600 <aa>
  40069d:       89 c6                   mov    %eax,%esi
  40069f:       bf 54 07 40 00          mov    $0x400754,%edi
  4006a4:       b8 00 00 00 00          mov    $0x0,%eax
  4006a9:       e8 12 fe ff ff          callq  4004c0 <printf@plt>
  4006ae:       bf a0 86 01 00          mov    $0x186a0,%edi
  4006b3:       b8 00 00 00 00          mov    $0x0,%eax
  4006b8:       e8 33 fe ff ff          callq  4004f0 <sleep@plt>
  4006bd:       5d                      pop    %rbp
  4006be:       c3                      retq   
  4006bf:       90                      nop
```

和例子1不同，aa函数中并没有显式调用bb函数的地方，而0x400672处有点像跳转的意思，我们在此设置断点：
```bash
$b *0x400672
$run
Breakpoint 1, 0x0000000000400672 in aa ()
(gdb) info registers 
rax            0x7fffffffe3d4   140737488348116   <<<<<<<<<<
rbx            0x0      0
rcx            0x1a     26
rdx            0xa      10
rsi            0x7fffffe5       2147483621
```
此时rax为0x7fffffffe3d4，显然该地址为栈地址，既然为跳转指令，我们可以看看这其中是在干什么：
```c
(gdb) disassemble 0x7fffffffe3d4+0x0, 0x7fffffffe3d4+0x20
Dump of assembler code from 0x7fffffffe3d4 to 0x7fffffffe3f4:
   0x00007fffffffe3d4:  mov    $0x4005ed,%r11d               <<<<<<
   0x00007fffffffe3da:  movabs $0x7fffffffe3d0,%r10          <<<<<<
   0x00007fffffffe3e4:  rex.WB jmpq *%r11                    <<<<<<
   0x00007fffffffe3e7:  nop
   0x00007fffffffe3e8:  or     $0x4007,%eax
   0x00007fffffffe3ed:  add    %al,(%rax)
   0x00007fffffffe3ef:  add    %ah,(%rax)
   0x00007fffffffe3f1:  in     $0xff,%al
   0x00007fffffffe3f3:  (bad)  
End of assembler dump.
```
1. 加载$0x4005ed到r11
2. 加载$0x7fffffffe3d0到r10
3. 跳转至r11

$0x4005ed就是bb函数的地址（00000000004005ed <bb.2180> ），$0x7fffffffe3d0应该是传入的参数
而aa中的这几段汇编就是在制造Trampoline：
```c
  400636:       66 c7 40 06 49 ba       movw   $0xba49,0x6(%rax)         <<<<<<<<<<<<<<<
  40063c:       48 89 50 08             mov    %rdx,0x8(%rax)            <<<<<<<<<<<<<<<
  400640:       c7 40 10 49 ff e3 90    movl   $0x90e3ff49,0x10(%rax)    <<<<<<<<<<<<<<<
```

很显然在x86的机器上该写法能够成功编译，并使用trampoline技术，运行正确，而vendor spec mips-gcc由于功能限制无法正确编译，包括一些嵌入式ARM体系结构的gcc（本人实测过）也不能支持此种内嵌函数的写法，所以解决方法就是走常规路子，将内嵌函数搬到外面来。

## 题外
Trampoline有什么好处呢，其最大的作用可以实现高级语言中支持的闭包，比如Python：
```python
def make_adder(addend):
    def adder(augend):
        return augend + addend
    return adder
```