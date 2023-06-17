---
title: 02 BombLab
tags:
  - c
  - project
  - 汇编
  - gdb
categories:
  - course
  - csapp
  - lab
abbrlink: 767
date: 2023-06-12 21:06:36
---

# 资料

**代码地址：**[http://csapp.cs.cmu.edu/3e/bomb.tar](http://csapp.cs.cmu.edu/3e/bomb.tar)

**x 86-64 GDB 命令：**[http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf](http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf)
[gdbnotes-x86-64](gdbnotes-x86-64.pdf)

**说明：**[http://csapp.cs.cmu.edu/3e/bomblab.pdf](http://csapp.cs.cmu.edu/3e/bomblab.pdf)
[bomblab](bomblab.pdf)

**README：**[http://csapp.cs.cmu.edu/3e/README-bomblab](http://csapp.cs.cmu.edu/3e/README-bomblab)
[README](./README.md)

# 基本工具

GDB
- 查阅 GDB 手册
- `man gdb`
- `info gdb`
- gdb `help`

`objdump -t`
- 打印符号表
- 包括函数名、全局变量、调用的函数和地址
- `objdump -t bomb>table` 将可执行文件 bomb 的符号表保存到 table 中。

`objdump -d`
-  反汇编可执行程序
- 可以查看具体函数的汇编代码，但是系统函数比如 `sscanf` 在汇编代码中会以一种隐晦的方式出现。这时候就需要用 GDB 反汇编去识别

# Lab 的基本流程

## 相关文件

解压代码文件，获得主要文件如下：

```sh
├── bomb
├── bomb.c
└── README
```

## 可执行文件

Lab 包括六个 phase，每个 phase 读入六个字符串，如果字符串正确就不会爆炸。

可以将答案预先写入自己创建的文件 `ans` 中，然后执行：

```sh
./bomb ans
```

## 源码

打开 `bomb. c` 可以看到包含 `main` 函数的代码文件：

```c
#include <stdio.h>
#include <stdlib.h>
#include "support.h"
#include "phases.h"

FILE *infile;

int main(int argc, char *argv[])
{
	...

    /* Hmm...  Six phases must be more secure than one phase! */
    input = read_line();             /* Get input                   */
    phase_1(input);                  /* Run the phase               */
    phase_defused();                 /* Drat!  They figured it out!
				      * Let me know how they did it. */
    printf("Phase 1 defused. How about the next one?\n");

    /* The second phase is harder.  No one will ever figure out
     * how to defuse this... */
    input = read_line();
    phase_2(input);
    phase_defused();
    printf("That's number 2.  Keep going!\n");

    /* I guess this is too easy so far.  Some more complex code will
     * confuse people. */
    input = read_line();
    phase_3(input);
    phase_defused();
    printf("Halfway there!\n");

    /* Oh yeah?  Well, how good is your math?  Try on this saucy problem! */
    input = read_line();
    phase_4(input);
    phase_defused();
    printf("So you got that one.  Try this one.\n");
    
    /* Round and 'round in memory we go, where we stop, the bomb blows! */
    input = read_line();
    phase_5(input);
    phase_defused();
    printf("Good work!  On to the next...\n");

    /* This phase will never be used, since no one will get past the
     * earlier ones.  But just in case, make this one extra hard. */
    input = read_line();
    phase_6(input);
    phase_defused();
   
    return 0;
}
```

`main` 函数的主体包含六个 phase，都是读入一行字符串然后调用 `phase_x` 检查。如果字符串不满足条件就会引爆。但是我们没有 `phase_x` 内部的具体实现的代码，所以需要就使用 gdb 的对 `bomb` 可执行文件反汇编，通过阅读汇编代码推测源码的形式。

## 使用 gdb 调试方法

- 为了防止炸弹爆炸，不直接在命令行执行 `./bomb` ，而是通过 gdb，先在炸弹爆炸代码前打断点。
-  gdb 命令手册： [http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf](http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf)

总结本次 lab 常用的 gdb 命令：

**基本使用：**

- `gdb <file>` 对可执行文件启用 gdb
- `q` / `quit` `Ctrl-d` 退出 gdb 
- `r` / `run` 开始执行，可以在后面附加参数比如： `r ans` 将 ans 文件作为附加参数
- `k` / `kill` 终止程序
- `Enter` 直接回车会重复上一次的命令

**查看代码：**

- `l` / `list` 查看代码文件
- `disas` 反汇编当前环境执行的代码
- `disas 函数名/汇编指令地址`

**断点：**

- `b` / `break` 在函数名/行号/汇编指令地址处打断点
- `d` / `delete` 指定断点序号并删除断点
- `disable` / `enable` 禁用/启用断点
- `info b` 查看当前所有断点的情况

**执行代码：**

- `si` / `stepi` 执行下一条==汇编==指令，可以在后面跟数字 k 表示执行后 k 条指令。
- `ni` / `nexti` 执行下一条汇编指令，但会跳过函数调用
- `s` / `step` 执行下一条 C 语言指令
- `n` / `next`  执行下一条 C 语言指令，但会跳过函数调用
- `until k` 一直执行直到遇到序号为 k 的断点。
- `c` / `continue` 继续执行直到遇到下一个断点
- `finish` 执行完当前的==函数体==

**检查数据：**

`print`
- `/d` `/x` `/t` 十进制、十六进制、二进制
- 可以打印寄存器、某地址处的数据
- `*(int*) $0x400xxx` 以**整型**打印内存地址处的数据
- `(char *) $0x400xxx` 打印内存地址处的**字符串**

`x/<n/f/u> `
- `n` 内存单元的个数
- `f` 格式：x 16 进制。**d 十进制**。T 二进制。C 字符。**s 字符串**。I 指令。
- `u` 内存单元的大小：b 字节。H 半字。W 字。G
- 常用：
	- `x/s 地址` 打印字符串
	- `x/6dw 地址` 以十进制打印长度为 6 的整型数组

**查看信息：**

`info`
- `prog` 程序执行情况
- `b` 断点情况
- `f` 环境
- `r` 寄存器
- `s` 栈

## 实验前的准备

在 gdb 内先用 `b expolde_bomb` 在炸弹爆炸代码前打断点，然后依次给各个 phase 打断点，最后 `run ans` 执行代码。

```sh
(gdb) info b
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x000000000040143a <explode_bomb>
2       breakpoint     keep y   0x0000000000400ee0 <phase_1>
3       breakpoint     keep y   0x0000000000400efc <phase_2>
4       breakpoint     keep y   0x0000000000400f43 <phase_3>
5       breakpoint     keep y   0x000000000040100c <phase_4>
6       breakpoint     keep y   0x0000000000401062 <phase_5>
7       breakpoint     keep y   0x00000000004010f4 <phase_6>
```

# Phase 主体

每个 phase 的形式相同：

```c
input = read_line();
phase_1(input);
phase_defused();
```

对应汇编代码：

```
0x0000000000400e32 <+146>:   callq  0x40149e <read_line>
0x0000000000400e37 <+151>:   mov    %rax,%rdi
0x0000000000400e3a <+154>:   callq  0x400ee0 <phase_1>
0x0000000000400e3f <+159>:   callq  0x4015c4 <phase_defused>
```

也就是说在调用 `phase` 的函数前， `$rax` 和 `$rdi` 存放的是输入的数据，即在 ans 中存放的每一行字符串的地址。

# Phase 1

执行到 phase_1 的断点，得到反汇编代码如下：

```
Dump of assembler code for function phase_1:
=> 0x0000000000400ee0 <+0>:     sub    $0x8,%rsp
   0x0000000000400ee4 <+4>:     mov    $0x402400,%esi
   0x0000000000400ee9 <+9>:     call   0x401338 <strings_not_equal>
   0x0000000000400eee <+14>:    test   %eax,%eax
   0x0000000000400ef0 <+16>:    je     0x400ef7 <phase_1+23>
   0x0000000000400ef2 <+18>:    call   0x40143a <explode_bomb>
   0x0000000000400ef7 <+23>:    add    $0x8,%rsp
   0x0000000000400efb <+27>:    ret    
End of assembler dump.
```

查看一下寄存器的状态，发现 `rax` `rsi` `rdi` 存放的都是输入==字符串地址==

继续执行到 `call`
- 调用函数 `strings_not_equal` 第一个参数为地址 `$0x402400` ，第二个参数为输入字符串的地址。
- 最后当函数的返回结果为 0 时就不会爆炸，否则调用 `expolde_bomb` 炸弹爆炸。

进一步查看函数 `strings_not_equal` 的反汇编代码，看看它到底做了什么（其实从名字就可以看出是将判断两个字符串的相等关系）：

```
%rdi input_line
%esi 0x402400
------------------------
Dump of assembler code for function strings_not_equal:
=> 0x0000000000401338 <+0>:     push   %r12
   0x000000000040133a <+2>:     push   %rbp
   0x000000000040133b <+3>:     push   %rbx
   0x000000000040133c <+4>:     mov    %rdi,%rbx
   0x000000000040133f <+7>:     mov    %rsi,%rbp
   0x0000000000401342 <+10>:    call   0x40131b <string_length>
   0x0000000000401347 <+15>:    mov    %eax,%r12d
   0x000000000040134a <+18>:    mov    %rbp,%rdi
   0x000000000040134d <+21>:    call   0x40131b <string_length>
   0x0000000000401352 <+26>:    mov    $0x1,%edx
   0x0000000000401357 <+31>:    cmp    %eax,%r12d
   0x000000000040135a <+34>:    jne    0x40139b <strings_not_equal+99>
   0x000000000040135c <+36>:    movzbl (%rbx),%eax
   0x000000000040135f <+39>:    test   %al,%al
   0x0000000000401361 <+41>:    je     0x401388 <strings_not_equal+80>
   0x0000000000401363 <+43>:    cmp    0x0(%rbp),%al
   0x0000000000401366 <+46>:    je     0x401372 <strings_not_equal+58>
   0x0000000000401368 <+48>:    jmp    0x40138f <strings_not_equal+87>
   0x000000000040136a <+50>:    cmp    0x0(%rbp),%al
   0x000000000040136d <+53>:    nopl   (%rax)
   0x0000000000401370 <+56>:    jne    0x401396 <strings_not_equal+94>
   0x0000000000401372 <+58>:    add    $0x1,%rbx
   0x0000000000401376 <+62>:    add    $0x1,%rbp
   0x000000000040137a <+66>:    movzbl (%rbx),%eax
   0x000000000040137d <+69>:    test   %al,%al
   0x000000000040137f <+71>:    jne    0x40136a <strings_not_equal+50>
   0x0000000000401381 <+73>:    mov    $0x0,%edx
   0x0000000000401386 <+78>:    jmp    0x40139b <strings_not_equal+99>
   0x0000000000401388 <+80>:    mov    $0x0,%edx
   0x000000000040138d <+85>:    jmp    0x40139b <strings_not_equal+99>
   0x000000000040138f <+87>:    mov    $0x1,%edx
   0x0000000000401394 <+92>:    jmp    0x40139b <strings_not_equal+99>
   0 x 0000000000401396 <+94>:    mov    $0 x 1,%edx
   0x000000000040139b <+99>:    mov    %edx,%eax
   0x000000000040139d <+101>:   pop    %rbx
   0x000000000040139e <+102>:   pop    %rbp
   0x000000000040139f <+103>:   pop    %r12
   0x00000000004013a1 <+105>:   ret  
```

可以看出该函数的作用是判断两个字符串是否不等。

查看在地址 `$0x402400` 的字符串，就是答案：

```sh
(gdb) x/s $rsi
0x402400:       "Border relations with Canada have never been better."
```

# Phase 2

**phase 2 函数主体：**

```
Dump of assembler code for function phase_2:
   0x0000000000400efc <+0>:     push   %rbp
   0x0000000000400efd <+1>:     push   %rbx
   0x0000000000400efe <+2>:     sub    $0x28,%rsp
   0x0000000000400f02 <+6>:     mov    %rsp,%rsi
   0x0000000000400f05 <+9>:     call   0x40145c <read_six_numbers>
   0x0000000000400f0a <+14>:    cmpl   $0x1,(%rsp)
   0x0000000000400f0e <+18>:    je     0x400f30 <phase_2+52>
   0x0000000000400f10 <+20>:    call   0x40143a <explode_bomb>
   0 x 0000000000400 f 15 <+25>:    jmp    0 x 400 f 30 <phase_2+52>
   0x0000000000400f17 <+27>:    mov    -0x4(%rbx),%eax
   0x0000000000400f1a <+30>:    add    %eax,%eax
   0x0000000000400f1c <+32>:    cmp    %eax,(%rbx)
   0x0000000000400f1e <+34>:    je     0x400f25 <phase_2+41>
   0x0000000000400f20 <+36>:    call   0x40143a <explode_bomb>
   0x0000000000400f25 <+41>:    add    $0x4,%rbx
   0x0000000000400f29 <+45>:    cmp    %rbp,%rbx
   0x0000000000400f2c <+48>:    jne    0x400f17 <phase_2+27>
   0x0000000000400f2e <+50>:    jmp    0x400f3c <phase_2+64>
   0x0000000000400f30 <+52>:    lea    0x4(%rsp),%rbx
   0x0000000000400f35 <+57>:    lea    0x18(%rsp),%rbp
   0x0000000000400f3a <+62>:    jmp    0x400f17 <phase_2+27>
   0x0000000000400f3c <+64>:    add    $0x28,%rsp
   0x0000000000400f40 <+68>:    pop    %rbx
   0x0000000000400f41 <+69>:    pop    %rbp
   0x0000000000400f42 <+70>:    ret 
```

函数主体大致分成四步：
- <+2~+6> 在栈上分配 40 B 局部内存空间，然后将其地址传入 `$rsi`
- <+9> 调用函数 `read_six_number` 共有两个参数：
	- **第一个参数**是用户输入的字符串
	- **第二个参数**是栈上局部空间的地址。
 - <+14~+62> 对函数运行结果进行一些检查，不符合要求就爆炸
 - <+64~> 收尾，恢复栈空间，恢复寄存器原值

**read_six_numbers：**

先查看 `read_six_numbers` 的汇编代码，了解一下它具体做了什么：

```
Dump of assembler code for function read_six_numbers:
=> 0x000000000040145c <+0>:     sub    $0x18,%rsp
   0x0000000000401460 <+4>:     mov    %rsi,%rdx
   0x0000000000401463 <+7>:     lea    0x4(%rsi),%rcx
   0x0000000000401467 <+11>:    lea    0x14(%rsi),%rax
   0x000000000040146b <+15>:    mov    %rax,0x8(%rsp)
   0x0000000000401470 <+20>:    lea    0x10(%rsi),%rax
   0 x 0000000000401474 <+24>:    mov    %rax, (%rsp)
   0x0000000000401478 <+28>:    lea    0xc(%rsi),%r9
   0x000000000040147c <+32>:    lea    0x8(%rsi),%r8
   0x0000000000401480 <+36>:    mov    $0x4025c3,%esi
   0x0000000000401485 <+41>:    mov    $0x0,%eax
   0x000000000040148a <+46>:    call   0x400bf0 <__isoc99_sscanf@plt>
   0x000000000040148f <+51>:    cmp    $0x5,%eax
   0x0000000000401492 <+54>:    jg     0x401499 <read_six_numbers+61>
   0x0000000000401494 <+56>:    call   0x40143a <explode_bomb>
   0x0000000000401499 <+61>:    add    $0x18,%rsp
   0x000000000040149d <+65>:    ret 
```

该函数的主体大致包含三步
- <+0~+41>分配 24 B 栈空间，配置传入参数
- <+46> 调用标准库函数 `sscanf` 
	- 在 shell 中 `man sscanf` 可以看看该函数的输入参数情况
	- 第一个参数为**输入字符串**
	- 第二个参数为**模式串**，地址为 `$0x4025c3` ，在 gdb 内查看内容为 
	```
	(gdb) x/s 0x4025c3
	0x4025c3:       "%d %d %d %d %d %d" 
	``` 
	- 后续的参数为读取的数据，根据函数名 `read_six_number` 和模式串内容可以得知会读取 6 个**整型数据**。
	- 函数的结果为成功读取的数据**个数**
- <+51~+56> 检查结果，不符合要求就爆炸
	- <+51> 和 <+54> 可以看出当成功读取的数据个数>5 时才不会爆炸
- <+61> 收尾，恢复栈空间 

总结：该函数传入一个字符串和一个数组地址，通过 `sscanf` 读取 6 个整型数据到数组中

**重新回到 phase 2：**

根据对 `read_six_number` 的分析，我们直到我们输入数据的格式应为 6 个整型数据，那么现在的重点就是应该填写哪些数据。所以现在的重点是看一下 <+14~+62> 的指令。

```
Dump of assembler code for function phase_2:
   0x0000000000400efc <+0>:     push   %rbp
   0x0000000000400efd <+1>:     push   %rbx
   0x0000000000400efe <+2>:     sub    $0x28,%rsp
   0x0000000000400f02 <+6>:     mov    %rsp,%rsi
   0x0000000000400f05 <+9>:     call   0x40145c <read_six_numbers>
   0x0000000000400f0a <+14>:    cmpl   $0x1,(%rsp)
   0x0000000000400f0e <+18>:    je     0x400f30 <phase_2+52>
   0x0000000000400f10 <+20>:    call   0x40143a <explode_bomb>
   0x0000000000400f15 <+25>:    jmp    0x400f30 <phase_2+52>
   0x0000000000400f17 <+27>:    mov    -0x4(%rbx),%eax
   0x0000000000400f1a <+30>:    add    %eax,%eax
   0x0000000000400f1c <+32>:    cmp    %eax,(%rbx)
   0x0000000000400f1e <+34>:    je     0x400f25 <phase_2+41>
   0x0000000000400f20 <+36>:    call   0x40143a <explode_bomb>
   0x0000000000400f25 <+41>:    add    $0x4,%rbx
   0x0000000000400f29 <+45>:    cmp    %rbp,%rbx
   0x0000000000400f2c <+48>:    jne    0x400f17 <phase_2+27>
   0x0000000000400f2e <+50>:    jmp    0x400f3c <phase_2+64>
   0x0000000000400f30 <+52>:    lea    0x4(%rsp),%rbx
   0x0000000000400f35 <+57>:    lea    0x18(%rsp),%rbp
   0x0000000000400f3a <+62>:    jmp    0x400f17 <phase_2+27>
   0x0000000000400f3c <+64>:    add    $0x28,%rsp
   0x0000000000400f40 <+68>:    pop    %rbx
   0x0000000000400f41 <+69>:    pop    %rbp
   0x0000000000400f42 <+70>:    ret 
```

- 在执行为 `read_six_numbers` 后，读取了 6 个数据并存放到栈空间上，具体为： `rsp` 向后的 24 B 的空间上。 
- <+14，+18> 说明第一个数字应为 1
- <+27~+48>为循环主体，<+52~+62>为循环初始化
	- 要求后五个数字每个数字是前一个数字的 2 倍否则爆炸
 
得到六个数字应为 `1 2 4 8 16`

# Phase 3

```text
Dump of assembler code for function phase_3:
=> 0x0000000000400f43 <+0>:     sub    $0x18,%rsp
   0x0000000000400f47 <+4>:     lea    0xc(%rsp),%rcx
   0x0000000000400f4c <+9>:     lea    0x8(%rsp),%rdx
   0x0000000000400f51 <+14>:    mov    $0x4025cf,%esi
   0x0000000000400f56 <+19>:    mov    $0x0,%eax
   0x0000000000400f5b <+24>:    callq  0x400bf0 <__isoc99_sscanf@plt>
   0x0000000000400f60 <+29>:    cmp    $0x1,%eax
   0x0000000000400f63 <+32>:    jg     0x400f6a <phase_3+39>
   0x0000000000400f65 <+34>:    callq  0x40143a <explode_bomb>
   0x0000000000400f6a <+39>:    cmpl   $0x7,0x8(%rsp)
   0x0000000000400f6f <+44>:    ja     0x400fad <phase_3+106>
   0x0000000000400f71 <+46>:    mov    0x8(%rsp),%eax
   0x0000000000400f75 <+50>:    jmpq   *0x402470(,%rax,8)
   0x0000000000400f7c <+57>:    mov    $0xcf,%eax
   0x0000000000400f81 <+62>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f83 <+64>:    mov    $0x2c3,%eax
   0x0000000000400f88 <+69>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f8a <+71>:    mov    $0x100,%eax
   0x0000000000400f8f <+76>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f91 <+78>:    mov    $0x185,%eax
   0x0000000000400f96 <+83>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400f98 <+85>:    mov    $0xce,%eax
   0 x 0000000000400 f 9 d <+90>:    jmp    0 x 400 fbe <phase_3+123>
   0x0000000000400f9f <+92>:    mov    $0x2aa,%eax
   0x0000000000400fa4 <+97>:    jmp    0x400fbe <phase_3+123>
   0x0000000000400fa6 <+99>:    mov    $0x147,%eax
   0x0000000000400fab <+104>:   jmp    0x400fbe <phase_3+123>
   0x0000000000400fad <+106>:   callq  0x40143a <explode_bomb>
   0x0000000000400fb2 <+111>:   mov    $0x0,%eax
   0x0000000000400fb7 <+116>:   jmp    0x400fbe <phase_3+123>
   0x0000000000400fb9 <+118>:   mov    $0x137,%eax
   0x0000000000400fbe <+123>:   cmp    0xc(%rsp),%eax
   0x0000000000400fc2 <+127>:   je     0x400fc9 <phase_3+134>
   0x0000000000400fc4 <+129>:   callq  0x40143a <explode_bomb>
   0x0000000000400fc9 <+134>:   add    $0x18,%rsp
   0x0000000000400fcd <+138>:   retq   
```

函数主体大致分为 5 步：
- <+0~+19> 分配栈空间，设置输入参数
	- 参数为输入参数、模式串、栈空间两个地址
	- 查看一下模式串
	```
	(gdb) x/s $esi
	0x4025cf:       "%d %d"
	```
	- 说明函数读取两个整型数据到 `0x8($rsp)` 和 `0xc($rsp)`
- <+24~ +32> 调用 `sscanf` 并检查是否正确读取了两个整型数
- <+39~+44> 第一个数必须小于等于 7
- <+50> `*` 表明了这里是一个 `switch` 的跳转结构，在 `0x402470` 存放一个跳转表。根据跳转指令的形式可以知道一条表项长 8 B，我们可以依次打印处所有的跳转表项查看。
- < `switch` 的各个分支> 将 `$eax` 设置为各个不同的值
- <+123~> 比较 `$eax` 和第二个数字，如果相等就不会爆炸。

```
(gdb) x/xw 0x402470
0x402470:       0x00400f7c
(gdb) x/xw 0x402478
0x402478:       0x00400fb9
(gdb) x/xw 0x402480
0x402480:       0x00400f83
(gdb) x/xw 0x402488
0x402488:       0x00400f8a
(gdb) x/xw 0x402490
0x402490:       0x00400f91
(gdb) x/xw 0x402498
0x402498:       0x00400f98
(gdb) x/xw 0x4024a0
0x4024a0:       0x00400f9f
(gdb) x/xw 0x4024a8
0x4024a8:       0x00400fa6
(gdb) x/xw 0x4024b0
0x4024b0 <array.3449>:  0x7564616d
```

跳转表的地址从 `0x402470` 到 `0x4024a0` ，可以依次查看各个目的地址处的指令：

```
   0x0000000000400f7c <+57>:    mov    $0xcf,%eax
   0x0000000000400fb9 <+118>:   mov    $0x137,%eax
   0x0000000000400f83 <+64>:    mov    $0x2c3,%eax
   0x0000000000400f8a <+71>:    mov    $0x100,%eax
   0x0000000000400f91 <+78>:    mov    $0x185,%eax
   0x0000000000400f98 <+85>:    mov    $0xce,%eax
   0x0000000000400f9f <+92>:    mov    $0x2aa,%eax
   0x0000000000400fa6 <+99>:    mov    $0x147,%eax
```

所以这题是多解，第一个数字代表跳转表的不同表项，只要保证第一个数字小于等于 7 并且让第二个数字和对应位置的数字相同就行

简单起见，让第一个数字为 0，则跳转地址为 `0x400f7c` ，那么第二个数字就是 `0xcf` 对应十进制数为 207。

# Phase 4

**phase_4：**

```
Dump of assembler code for function phase_4:
=> 0x000000000040100c <+0>:     sub    $0x18,%rsp
   0x0000000000401010 <+4>:     lea    0xc(%rsp),%rcx
   0x0000000000401015 <+9>:     lea    0x8(%rsp),%rdx
   0x000000000040101a <+14>:    mov    $0x4025cf,%esi
   0x000000000040101f <+19>:    mov    $0x0,%eax
   0x0000000000401024 <+24>:    callq  0x400bf0 <__isoc99_sscanf@plt>
   0x0000000000401029 <+29>:    cmp    $0x2,%eax
   0x000000000040102c <+32>:    jne    0x401035 <phase_4+41>
   0x000000000040102e <+34>:    cmpl   $0xe,0x8(%rsp)
   0x0000000000401033 <+39>:    jbe    0x40103a <phase_4+46>
   0x0000000000401035 <+41>:    callq  0x40143a <explode_bomb>
   0x000000000040103a <+46>:    mov    $0xe,%edx
   0x000000000040103f <+51>:    mov    $0x0,%esi
   0x0000000000401044 <+56>:    mov    0x8(%rsp),%edi
   0x0000000000401048 <+60>:    callq  0x400fce <func4>
   0x000000000040104d <+65>:    test   %eax,%eax
   0x000000000040104f <+67>:    jne    0x401058 <phase_4+76>
   0x0000000000401051 <+69>:    cmpl   $0x0,0xc(%rsp)
   0x0000000000401056 <+74>:    je     0x40105d <phase_4+81>
   0x0000000000401058 <+76>:    callq  0x40143a <explode_bomb>
   0x000000000040105d <+81>:    add    $0x18,%rsp
   0x0000000000401061 <+85>:    retq 
```

函数主体大致分为：
- 分配栈上局部空间，设置输入参数
- 调用 `sscanf` ，和 phase 3 一样都是读取两个整型数到 `0x8($rsp)` 和 `0xc($rsp)`
- <+34~+39> 要求第一个数字小于等于 `0xe`
- <+46~+60> 调用 `func4` ，第一个参数为输入的第一个数字，第二个参数为 0，第三个参数为 `0xe`
- <+65~+67> 函数结果为 0 则不爆炸
- <+69~76> 第二个数字为 0 则不爆炸
- 恢复栈空间

这里可以确定第二个数字必须为 0，而第一个数字的取值取决于 `func4` 的函数结果。也可以直接试第一个数字的取值, 它的范围是 0 到 14。

**func 4：**

```
Dump of assembler code for function func4:
=> 0x0000000000400fce <+0>:     sub    $0x8,%rsp
   0x0000000000400fd2 <+4>:     mov    %edx,%eax
   0x0000000000400fd4 <+6>:     sub    %esi,%eax
   0x0000000000400fd6 <+8>:     mov    %eax,%ecx
   0x0000000000400fd8 <+10>:    shr    $0x1f,%ecx
   0x0000000000400fdb <+13>:    add    %ecx,%eax
   0x0000000000400fdd <+15>:    sar    %eax
   0x0000000000400fdf <+17>:    lea    (%rax,%rsi,1),%ecx
   0x0000000000400fe2 <+20>:    cmp    %edi,%ecx
   0x0000000000400fe4 <+22>:    jle    0x400ff2 <func4+36>
   0x0000000000400fe6 <+24>:    lea    -0x1(%rcx),%edx
   0x0000000000400fe9 <+27>:    callq  0x400fce <func4>
   0x0000000000400fee <+32>:    add    %eax,%eax
   0x0000000000400ff0 <+34>:    jmp    0x401007 <func4+57>
   0x0000000000400ff2 <+36>:    mov    $0x0,%eax
   0x0000000000400ff7 <+41>:    cmp    %edi,%ecx
   0x0000000000400ff9 <+43>:    jge    0x401007 <func4+57>
   0x0000000000400ffb <+45>:    lea    0x1(%rcx),%esi
   0x0000000000400ffe <+48>:    callq  0x400fce <func4>
   0x0000000000401003 <+53>:    lea    0x1(%rax,%rax,1),%eax
   0x0000000000401007 <+57>:    add    $0x8,%rsp
   0x000000000040100b <+61>:    retq   
End of assembler dump.
```

- <+27><+48> 又调用了 `func4` 说明此函数是一个递归函数。
- <+22><+43> 就是递归的退出条件

函数大致流程为：
- <+0~+20> 进行一系列运算, 简单起见设函数三个参数为 **a, b, c**：**d=((c-b)+(c-b)>>0 x 1 f)>>1 + b**
- <+22> **d <= a**？
	- 不满足要求则<+27>递归 `func4(a,b,d-1)` ，返回值×2，然后函数结束
	- 满足要求则进入<+43> 下一个递归退出条件 **a >= d?**
		- 满足要求<+57>则函数结束, 返回 (c-a)/2
		- 不满足要求则再<+48>递归 `func4(a,d+1,c)` ，返回值×2+1

> `>>0x1f` ：正数时为 0，负数时为 1。

# Phase 5

```text
Dump of assembler code for function phase_5:
   0x0000000000401062 <+0>:     push   %rbx
   0x0000000000401063 <+1>:     sub    $0x20,%rsp
   0x0000000000401067 <+5>:     mov    %rdi,%rbx
   0x000000000040106a <+8>:     mov    %fs:0x28,%rax
   0x0000000000401073 <+17>:    mov    %rax,0x18(%rsp)
=> 0x0000000000401078 <+22>:    xor    %eax,%eax
   0x000000000040107a <+24>:    callq  0x40131b <string_length>
   0x000000000040107f <+29>:    cmp    $0x6,%eax
   0x0000000000401082 <+32>:    je     0x4010d2 <phase_5+112>
   0x0000000000401084 <+34>:    callq  0x40143a <explode_bomb>
   0x0000000000401089 <+39>:    jmp    0x4010d2 <phase_5+112>
   0x000000000040108b <+41>:    movzbl (%rbx,%rax,1),%ecx
   0x000000000040108f <+45>:    mov    %cl,(%rsp)
   0x0000000000401092 <+48>:    mov    (%rsp),%rdx
   0x0000000000401096 <+52>:    and    $0xf,%edx
   0x0000000000401099 <+55>:    movzbl 0x4024b0(%rdx),%edx
   0x00000000004010a0 <+62>:    mov    %dl,0x10(%rsp,%rax,1)
   0x00000000004010a4 <+66>:    add    $0x1,%rax
   0x00000000004010a8 <+70>:    cmp    $0x6,%rax
   0x00000000004010ac <+74>:    jne    0x40108b <phase_5+41>
   0x00000000004010ae <+76>:    movb   $0x0,0x16(%rsp)
   0x00000000004010b3 <+81>:    mov    $0x40245e,%esi
   0x00000000004010b8 <+86>:    lea    0x10(%rsp),%rdi
   0x00000000004010bd <+91>:    callq  0x401338 <strings_not_equal>
   0x00000000004010c2 <+96>:    test   %eax,%eax
   0x00000000004010c4 <+98>:    je     0x4010d9 <phase_5+119>
   0x00000000004010c6 <+100>:   callq  0x40143a <explode_bomb>
   0x00000000004010cb <+105>:   nopl   0x0(%rax,%rax,1)
   0x00000000004010d0 <+110>:   jmp    0x4010d9 <phase_5+119>
   0x00000000004010d2 <+112>:   mov    $0x0,%eax
   0x00000000004010d7 <+117>:   jmp    0x40108b <phase_5+41>
   0x00000000004010d9 <+119>:   mov    0x18(%rsp),%rax
   0x00000000004010de <+124>:   xor    %fs:0x28,%rax
   0x00000000004010e7 <+133>:   je     0x4010ee <phase_5+140>
   0x00000000004010e9 <+135>:   callq  0x400b30 <__stack_chk_fail@plt>
   0x00000000004010ee <+140>:   add    $0x20,%rsp
   0x00000000004010f2 <+144>:   pop    %rbx
   0x00000000004010f3 <+145>:   retq 
```

+8、+17：在%rsp+24 处设置金丝雀值

+29、+32：输入的字符串长度应为 6

+41~+74 循环
- +41 : %rbx 存储输入字符串的首地址，%rax 存储 i ，也就是说每次循环读取字符串的一个字符
- +48：字符低 8 位存储在 (%rsp)上
- +52：将字符 and 0 xf （取低四位）结果设为**j**
- +55 : 有个全局数组 A 地址为 `0x4024b0` ,访问 A+ j 处的字符并存放在%edx 中
	- *全局数组内容 maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?*
- 将全局字符串的字符放在 16+%rsp+%rax 中
+76 循环结束字符串末尾添加结束符
+81 *全局字符串 0 x 40245 e "flyers"*
+91 判断字符串是否相等，相等则拆弹成功
+119 检查金丝雀值是否被破坏

总结：
根据输入的字符串，每个字符取末尾 4 位，分别按照结果查询全局字符串 0 x 4024 b 0 的对应位置的字符并组成字符串，然后将判断结果是否等于全局字符串 0 x 40245 e "flyers"。

一种结果为：9/. 567

# Phase 6

```text
Dump of assembler code for function phase_6:
   0x00000000004010f4 <+0>:     push   %r14
=> 0x00000000004010f6 <+2>:     push   %r13
   0x00000000004010f8 <+4>:     push   %r12
   0x00000000004010fa <+6>:     push   %rbp
   0x00000000004010fb <+7>:     push   %rbx
   0x00000000004010fc <+8>:     sub    $0x50,%rsp
   0x0000000000401100 <+12>:    mov    %rsp,%r13
   0x0000000000401103 <+15>:    mov    %rsp,%rsi
   0x0000000000401106 <+18>:    callq  0x40145c <read_six_numbers>
   0x000000000040110b <+23>:    mov    %rsp,%r14
   0x000000000040110e <+26>:    mov    $0x0,%r12d
   0x0000000000401114 <+32>:    mov    %r13,%rbp
   0x0000000000401117 <+35>:    mov    0x0(%r13),%eax
   0x000000000040111b <+39>:    sub    $0x1,%eax
   0x000000000040111e <+42>:    cmp    $0x5,%eax
   0x0000000000401121 <+45>:    jbe    0x401128 <phase_6+52>
   0x0000000000401123 <+47>:    callq  0x40143a <explode_bomb>
   0x0000000000401128 <+52>:    add    $0x1,%r12d
   0x000000000040112c <+56>:    cmp    $0x6,%r12d
   0x0000000000401130 <+60>:    je     0x401153 <phase_6+95>
   0x0000000000401132 <+62>:    mov    %r12d,%ebx
   0x0000000000401135 <+65>:    movslq %ebx,%rax
   0x0000000000401138 <+68>:    mov    (%rsp,%rax,4),%eax
   0x000000000040113b <+71>:    cmp    %eax,0x0(%rbp)
   0x000000000040113e <+74>:    jne    0x401145 <phase_6+81>
   0x0000000000401140 <+76>:    callq  0x40143a <explode_bomb>
   0x0000000000401145 <+81>:    add    $0x1,%ebx
   0x0000000000401148 <+84>:    cmp    $0x5,%ebx
   0x000000000040114b <+87>:    jle    0x401135 <phase_6+65>
   0x000000000040114d <+89>:    add    $0x4,%r13
   0x0000000000401151 <+93>:    jmp    0x401114 <phase_6+32>
   0x0000000000401153 <+95>:    lea    0x18(%rsp),%rsi
   0x0000000000401158 <+100>:   mov    %r14,%rax
   0x000000000040115b <+103>:   mov    $0x7,%ecx
   0x0000000000401160 <+108>:   mov    %ecx,%edx
   0x0000000000401162 <+110>:   sub    (%rax),%edx
   0x0000000000401164 <+112>:   mov    %edx,(%rax)
   0x0000000000401166 <+114>:   add    $0x4,%rax
   0x000000000040116a <+118>:   cmp    %rsi,%rax
   0x0000000000401176 <+130>:   mov    0x8(%rdx),%rdx
   0x000000000040117a <+134>:   add    $0x1,%eax
   0x000000000040117d <+137>:   cmp    %ecx,%eax
   0x000000000040117f <+139>:   jne    0x401176 <phase_6+130>
   0x0000000000401181 <+141>:   jmp    0x401188 <phase_6+148>
   0x0000000000401183 <+143>:   mov    $0x6032d0,%edx
   0x0000000000401188 <+148>:   mov    %rdx,0x20(%rsp,%rsi,2)
   0x000000000040118d <+153>:   add    $0x4,%rsi
   0x0000000000401191 <+157>:   cmp    $0x18,%rsi
   0x0000000000401195 <+161>:   je     0x4011ab <phase_6+183>
   0x0000000000401197 <+163>:   mov    (%rsp,%rsi,1),%ecx
   0x000000000040119a <+166>:   cmp    $0x1,%ecx
   0 x 000000000040119 d <+169>:   jle    0 x 401183 <phase_6+143>
   0x000000000040119f <+171>:   mov    $0x1,%eax
   0 x 00000000004011 a 4 <+176>:   mov    $0 x 6032 d 0,%edx
   0x00000000004011a9 <+181>:   jmp    0x401176 <phase_6+130>
   0x00000000004011ab <+183>:   mov    0x20(%rsp),%rbx
   0x00000000004011b0 <+188>:   lea    0x28(%rsp),%rax
   0x00000000004011b5 <+193>:   lea    0x50(%rsp),%rsi
   0x00000000004011ba <+198>:   mov    %rbx,%rcx
   0x00000000004011bd <+201>:   mov    (%rax),%rdx
   0x00000000004011c0 <+204>:   mov    %rdx,0x8(%rcx)
   0x00000000004011c4 <+208>:   add    $0x8,%rax
   0x00000000004011c8 <+212>:   cmp    %rsi,%rax
   0x00000000004011cb <+215>:   je     0x4011d2 <phase_6+222>
   0x00000000004011cd <+217>:   mov    %rdx,%rcx
   0x00000000004011d0 <+220>:   jmp    0x4011bd <phase_6+201>
   0x00000000004011d2 <+222>:   movq   $0x0,0x8(%rdx)
   0x00000000004011da <+230>:   mov    $0x5,%ebp
   0x00000000004011df <+235>:   mov    0x8(%rbx),%rax
   0x00000000004011e3 <+239>:   mov    (%rax),%eax
   0x00000000004011e5 <+241>:   cmp    %eax,(%rbx)
   0x00000000004011e7 <+243>:   jge    0x4011ee <phase_6+250>
   0x00000000004011e9 <+245>:   callq  0x40143a <explode_bomb>
   0x00000000004011ee <+250>:   mov    0x8(%rbx),%rbx
   0x00000000004011f2 <+254>:   sub    $0x1,%ebp
   0x00000000004011f5 <+257>:   jne    0x4011df <phase_6+235>
   0x00000000004011f7 <+259>:   add    $0x50,%rsp
   0x00000000004011fb <+263>:   pop    %rbx
   0x00000000004011fc <+264>:   pop    %rbp
   0x00000000004011fd <+265>:   pop    %r12
   0x00000000004011ff <+267>:   pop    %r13
   0x0000000000401201 <+269>:   pop    %r14
```

<+18>: 读取 6 个数
<+23>: 保存 rsp 到 r14

<+65>~<+87>: 内层循环，判断数与其他 5 个数是否是不同的，如果相同则爆炸
<+32>~<+93>: 外层循环，一次判断所有 6 个数

<+108>~<+121>: 循环， `a[i]=7-a[i]`

<+143>~<+169>: 循环，由<+128>进入，如若 `a[i]>1` 或遍历完 6 个数则退出循环
- 有 `a[i]>1`  :<+171>: 保存地址 `0x6032d0` 跳转到循环<+130>~<+139>:
- `a[i]<=1` 均成立：<+183>:

<+130>~<+139>: