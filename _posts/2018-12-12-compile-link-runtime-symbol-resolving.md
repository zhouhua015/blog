---
layout: post
title: 运行时的符号决议
categories:
  - c/cpp
categories:
  - details
  - linux
  - runtime
date: 2018-12-12
---

时至今日的软件规模，使用动态库进行代码封装复用已成行业标准。公共代码通过单独的开发编译链接形成动态库，在整个软件的其他部分被反复使用，并且随产品安装包分发到生产环境，成为运行时的一部分。对于现代 `Linux` 系统，动态库和可执行程序均为 [ELF](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 格式。

## ELF 文件
![ELF file layout](https://upload.wikimedia.org/wikipedia/commons/thumb/7/77/Elf-layout--en.svg/800px-Elf-layout--en.svg.png?1544616256637)

用命令行工具 `readelf` 可以查看文件内容，比如 `libc.so.6`：
``` shell
$ readelf -a /lib64/libc.so.6
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 03 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - GNU
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x21d10
  Start of program headers:          64 (bytes into file)
  Start of section headers:          2122536 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         10
  Size of section headers:           64 (bytes)
  Number of section headers:         75
  Section header string table index: 74

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .note.gnu.build-i NOTE             0000000000000270  00000270
       0000000000000024  0000000000000000   A       0     0     4
  [ 2] .note.ABI-tag     NOTE             0000000000000294  00000294
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .gnu.hash         GNU_HASH         00000000000002b8  000002b8
       0000000000003a70  0000000000000000   A       4     0     8
  [ 4] .dynsym           DYNSYM           0000000000003d28  00003d28
       000000000000d050  0000000000000018   A       5     3     8
  [ 5] .dynstr           STRTAB           0000000000010d78  00010d78
       00000000000058b5  0000000000000000   A       0     0     1
  ......

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x0000000000000230 0x0000000000000230  R E    8
  INTERP         0x00000000001849a0 0x00000000001849a0 0x00000000001849a0
                 0x000000000000001c 0x000000000000001c  R      10
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x00000000001b7d94 0x00000000001b7d94  R E    200000
  LOAD           0x00000000001b8730 0x00000000003b8730 0x00000000003b8730
                 0x0000000000005170 0x0000000000009a90  RW     200000
  DYNAMIC        0x00000000001bbb80 0x00000000003bbb80 0x00000000003bbb80
                 0x00000000000001f0 0x00000000000001f0  RW     8
  ......
```

## 程序启动
静态程序的启动相对简单，内核将 elf 文件[加载](https://lwn.net/Articles/631631/)到内存后，调用入口函数即可。而对于动态链接程序（当今大部分程序）而言，在将进程控制权交给入口函数前，必须加载所有依赖的动态库文件并决议所有未定义的符号。这些动态链接和符号决议的过程由动态链接器完成，其具体位置由 `Program Header` 中的 `INTERP` 条目指定。
``` shell
Program Headers:
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align

  ......

  INTERP         0x00000000001849a0 0x00000000001849a0 0x00000000001849a0
                 0x000000000000001c 0x000000000000001c  R      10
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```
`/lib64/ld-linux-x86-64.so.2` 的职责包括
* 加载可执行程序和所有依赖项
* 动态链接依赖项
* 初始化

符号决议就发生在动态链接依赖项阶段，所有在静态编译链接期间用到但未明确的符号都需要在此阶段重新定位。

`/lib64/ld-linux-x86-64.so.2` 默认延迟决议，即在符号第一次被使用时进行真正的定位操作。设置环境变量 `LD_BIND_NOW` 为任意非空内容，可以强制链接器在启动阶段完成所有符号决议。

不论是延迟决议还是立即决议，所有动态库的所有待决议符号都通过同一个流程完成定位，总是从最起点开始启动定位流程，总是涉及到两个核心概念：`Global Offset Table` 和 `Procedure Linkable Table`

## GOT 和 PLT

``` c
extern int foo;
extern void bar(int);

void call_bar_with_foo() {
    bar(foo);
}
```

这段代码编译生成动态库会产生两个需要运行时决议的符号，外部变量 `foo` 和函数 `bar`

``` disassembly
0000000000000bb0 <bar@plt-0x10>:
 bb0:   ff 35 52 14 20 00       pushq  0x201452(%rip)        # 202008 <_GLOBAL_OFFSET_TABLE_+0x8>
 bb6:   ff 25 54 14 20 00       jmpq   *0x201454(%rip)        # 202010 <_GLOBAL_OFFSET_TABLE_+0x10>
 bbc:   0f 1f 40 00             nopl   0x0(%rax)

0000000000000bc0 <bar@plt>:
 bc0:   ff 25 52 14 20 00       jmpq   *0x201452(%rip)        # 202018 <_GLOBAL_OFFSET_TABLE_+0x18>
 bc6:   68 00 00 00 00          pushq  $0x0
 bcb:   e9 e0 ff ff ff          jmpq   bb0 <_init+0x28>

....

movl    foo@got(%ebx), %eax
pushl   (%eax)
call    bar@plt

...

Disassembly of section .got.plt:

0000000000202000 <_GLOBAL_OFFSET_TABLE_>:
  202000:       d0 1d 20 00 00 00       rcrb   0x20(%rip)        # 202026 <_GLOBAL_OFFSET_TABLE_+0x26>
        ...
  202016:       00 00                   add    %al,(%rax)
  202018:       c6                      (bad)
  202019:       0b 00                   or     (%rax),%eax
  20201b:       00 00                   add    %al,(%rax)
  20201d:       00 00                   add    %al,(%rax)
  20201f:       00 d6                   add    %dl,%dh
```

从生成的汇编代码可以看到，外部变量 `foo` 变成 `foo@got`，而函数 `bar` 现在是 `bar@plt`。

查看 `bar@plt` 符号的内容，第一条指令跳转到 202018 处，此处地址则正好指向紧随其后的第二条指令（`bc6`处），在压栈偏移量后，执行 `bar@plt-0x10` 的指令。控制流程随后开始压栈更多参数，并调用链接器开始决议过程。当符号决议完成后，链接器修改 `got` 偏移量 `0x18` 处的值，写入真正的符号地址。自此以后，`bar@plt` 调用即可直接跳转到真正的符号地址，无需二次决议。

通过动态符号决议，特别是利用 `GOT` 和 `PLT`，动态库一经发布，即可被重复利用。系统当前已加载的动态库，其他进程再次使用时也无需重复读取。至此，动态库代码封装和复用的目标圆满达成，大规模软件开发协作由此成为可能。
