---
title: 函数栈大小分析
date: 2021-06-12 20:25:49
tags: stack, frame, arm, linux
---

## 程序运行

[原始文档](https://www.embeddedrelated.com/showarticle/1330.php)

### 基本知识

#### 内存

以下假设内存空间类似一个梯子，从上到下，地址值从小到大。

程序运行时内存主要分3种区域：

* 静态内存，存储全局变量，静态变量 即BSS和Data段
* 堆，malloc动态分配的内存，使用free释放
* 栈，函数调用过程中动态分配的内存段。每个函数有自己的栈帧，包括函数的局部变量和返回值信息。可以通过alloca函数扩展当前栈帧。

这三个区域在系统中的大小是预设好的，需要根据应用的情况进行分配各个区域的大小。如果一个区域分配的不合理，可能出现堆空间耗尽或栈溢出(stackoverflow)

#### ARM 汇编学习

https://azeria-labs.com/writing-arm-assembly-part-1/

#### 问题

作者发现`getaddrinfo()`  在他的树莓派系统初始化过程中占用了大量的栈空间，所以写了一个测试程序

```c
#include <sys/socket.h>
#include <netdb.h>
#include <string.h>
int main()
{
    struct addrinfo hints;
    struct addrinfo* address_list;
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_protocol = IPPROTO_TCP;
    int result = getaddrinfo("test.example.com", "80", &hints, &address_list);
    return result;
}
```

编译运行

`gcc test-getaddrinfo.c -o test-getaddrinfo -g `

#### 分析过程

查看Linux给程序分配的栈开始和结束位置

`/proc/<pid>/maps`文件中列出了内存的所有分段，`/proc/`文件系统可以看作是查看内核数据的一个UI界面。

也可以在一个运行的gdb会话中执行`info proc map`

对于Nucleo的实时系统，这个地址区间可能在他的链接控制脚本(.ld)文件中

Linux中的栈使用从高地址向低地址方向，即从End到Start的方向使用。

一个栈帧包含了函数运行需要的所有信息，例如暂时保存寄存器中的值，局部变量，函数参数。ARM EABI (Embedded Application Binary Interfac)规定函数的第一个参数通过寄存器传递。

栈区域在进程创建时全部初始化为0.所以可以从栈的开始地址找第一个值为非0的地址，就可以找到当前程序执行的栈的最大深度（从栈底到栈顶的长度）

 SP (Stack Pointer)当前栈顶指针，gdb中对应变量$sp

FP (Frame Pointer)当前栈帧地址，gdb中对应变量$r11

函数调用时，通过对SP的值进行减法操作（从高地址向低地址使用），例如当前函数执行需要20字节空间，就对`sp=sp-20`，让sp指向当前栈空间的顶部。这个操作只是移动了sp指向的位置，对其中的内存并没有执行初始化，所以如果对函数的局部变量不进行初始化就使用，局部变量的值可能就是原来这个内存区域的值，很有可能造成bug。

##### gdb调试程序

`-q`选项去掉gdb的启动信息 `gdb -q ./test-getaddrinfo`

使用`(gdb) list`命令查看当前的源代码

```c
1 #include <sys/socket.h>
2 #include <netdb.h>
3 #include <string.h>
4 
5 int
6 main()
7 {
8     struct addrinfo  hints;
9     struct addrinfo* address_list;
10 
11     memset(&hints, 0, sizeof(hints));
12     hints.ai_family   = AF_UNSPEC;
13     hints.ai_socktype = SOCK_STREAM;
14     hints.ai_protocol = IPPROTO_TCP;
15 
16     int result = getaddrinfo("test.example.com", "80", &hints, &address_list);
17     return result;
18 }
```

在main函数打断点 `(gdb) b main`

在main返回之前的17行打断点`(gdb) b 17`

开始运行程序`(gdb) r`

在程序在main中断点停止后，查看栈地址信息`(gdb) info proc map`

```c
process 10163
Mapped address spaces:
 Start Addr   End Addr       Size     Offset objfile
    0x10000    0x11000     0x1000        0x0 /home/pi/Projects/test-getaddrinfo/test-getaddrinfo
    0x20000    0x21000     0x1000        0x0 /home/pi/Projects/test-getaddrinfo/test-getaddrinfo
    0x21000    0x22000     0x1000     0x1000 /home/pi/Projects/test-getaddrinfo/test-getaddrinfo
 0x76e64000 0x76f8e000   0x12a000        0x0 /lib/arm-linux-gnueabihf/libc-2.24.so
 0x76f8e000 0x76f9d000     0xf000   0x12a000 /lib/arm-linux-gnueabihf/libc-2.24.so
 0x76f9d000 0x76f9f000     0x2000   0x129000 /lib/arm-linux-gnueabihf/libc-2.24.so
 0x76f9f000 0x76fa0000     0x1000   0x12b000 /lib/arm-linux-gnueabihf/libc-2.24.so
 0x76fa0000 0x76fa3000     0x3000        0x0 
 0x76fb8000 0x76fbd000     0x5000        0x0 /usr/lib/arm-linux-gnueabihf/libarmmem.so
 0x76fbd000 0x76fcc000     0xf000     0x5000 /usr/lib/arm-linux-gnueabihf/libarmmem.so
 0x76fcc000 0x76fcd000     0x1000     0x4000 /usr/lib/arm-linux-gnueabihf/libarmmem.so
 0x76fcd000 0x76fce000     0x1000     0x5000 /usr/lib/arm-linux-gnueabihf/libarmmem.so
 0x76fce000 0x76fef000    0x21000        0x0 /lib/arm-linux-gnueabihf/ld-2.24.so
 0x76ff9000 0x76ffb000     0x2000        0x0 
 0x76ffb000 0x76ffc000     0x1000        0x0 [sigpage]
 0x76ffc000 0x76ffd000     0x1000        0x0 [vvar]
 0x76ffd000 0x76ffe000     0x1000        0x0 [vdso]
 0x76ffe000 0x76fff000     0x1000    0x20000 /lib/arm-linux-gnueabihf/ld-2.24.so
 0x76fff000 0x77000000     0x1000    0x21000 /lib/arm-linux-gnueabihf/ld-2.24.so
 0x7efdf000 0x7f000000    0x21000        0x0 [stack]
 0xffff0000 0xffff1000     0x1000        0x0 [vectors]
```

可以看到栈的结束位置在`0x7f000000`，大小为`0x21000`，可以算出来栈的开始位置为`0x7EFDF000`

 **注意：**这里的栈大小不是Linux系统默认的8M，是132K，这是系统默认给当前进程分配的大小，当进程中的使用的栈空间更多时，系统会扩大这个区域的大小。例如在一个函数中使用了2M的局部变量，系统会把stack区域范围调大，即把低地址0x7efdf000再像低地址区域扩大，例如编程0x7bf00000

查看当前栈执行最大深度

```bash
(gdb) scan_stack 0 $stack_size
Scanned 10000
Scanned 20000
Scanned 30000
Scanned 40000
Scanned 50000
Scanned 60000
Scanned 70000
Scanned 80000
Scanned 90000
Scanned 100000
Scanned 110000
Scanned 120000
Found data 4660 bytes deeper than current stack frame (0x7effeeb0).
Address    2130697340 = 0x7effdc7c
Stack size   135168 = 0x21000 = 132.0KB, 0x7efdf000-0x7f000000
Stack offset 126076 = 0x1ec7c = 123.1KB
Stack depth    9092 = 0x02384 =   8.9KB
0x7effdc7c: 0x00000020 0x00002e41 0x61656100 0x01006962
0x7effdc8c: 0x00000024 0x06003605 0x09010806 0x12020a01
0x7effdc9c: 0x14011304 0x16011501 0x18031701 0x1c021a01
0x7effdcac: 0x00012201 0x00000000 0x7effe8f4 0x00000000
```

可以出当前使用栈的最大深度是8.9K，而栈顶的历史最大值比当前SP的值还小了4660字节。这是因为系统在执行我们的程序的main函数之前进行的库和数据段的初始化，例如把二进制程序中的`.data`段数据拷贝到静态内存区域，初始化全局变量和静态变量。

查看当前栈顶的深度

```bash
(gdb) stack_offset $sp
Address    2130702000 = 0x7effeeb0
Stack size   135168 = 0x21000 = 132.0KB, 0x7efdf000-0x7f000000
Stack offset 130736 = 0x1feb0 = 127.7KB
Stack depth    4432 = 0x01150 =   4.3KB
```

查看当前程序的汇编

```asm
(gdb) disassemble 
Dump of assembler code for function main:
   0x00010474 <+0>: push {r11, lr}
   0x00010478 <+4>: add r11, sp, #4
   0x0001047c <+8>: sub sp, sp, #40 ; 0x28
=> 0x00010480 <+12>: sub r3, r11, #40 ; 0x28
   0x00010484 <+16>: mov r2, #32
   0x00010488 <+20>: mov r1, #0
   0x0001048c <+24>: mov r0, r3
   0x00010490 <+28>: bl 0x10328 <memset@plt>
   0x00010494 <+32>: mov r3, #0
   0x00010498 <+36>: str r3, [r11, #-36] ; 0xffffffdc
   0x0001049c <+40>: mov r3, #1
   0x000104a0 <+44>: str r3, [r11, #-32] ; 0xffffffe0
   0x000104a4 <+48>: mov r3, #6
   0x000104a8 <+52>: str r3, [r11, #-28] ; 0xffffffe4
   0x000104ac <+56>: sub r3, r11, #44 ; 0x2c
   0x000104b0 <+60>: sub r2, r11, #40 ; 0x28
   0x000104b4 <+64>: ldr r1, [pc, #24] ; 0x104d4 <main+96>
   0x000104b8 <+68>: ldr r0, [pc, #24] ; 0x104d8 <main+100>
   0x000104bc <+72>: bl 0x10334 <getaddrinfo@plt>
   0x000104c0 <+76>: str r0, [r11, #-8]
   0x000104c4 <+80>: ldr r3, [r11, #-8]
   0x000104c8 <+84>: mov r0, r3
   0x000104cc <+88>: sub sp, r11, #4
   0x000104d0 <+92>: pop {r11, pc}
   0x000104d4 <+96>: andeq r0, r1, r12, asr #10
   0x000104d8 <+100>: andeq r0, r1, r0, asr r5
End of assembler dump.
```

如果我们有当前程序的源代码，可以匹配使用`(gdb) disassemble /s`匹配到源代码

每一个函数的汇编由序言，正文和结尾组成，序言用来保存返回上一个函数的地址以及分配当前函数的栈帧空间，正文是函数内容的实现，结尾返回值并跳转回上一级地址。

* ARM汇编函数的序言

```
0x00010474 <+0>: push {r11, lr}
0x00010478 <+4>: add r11, sp, #4
0x0001047c <+8>: sub sp, sp, #40 ; 0x28
```

1. 把当前FP和LR(Link Register)这两个寄存器的值依次压入栈中，LR中是上一级函数中调用当前函数后的下一个指令地址
2. 把SP的值+4，然后把结果存入FP中，此时FP指向的是当前栈帧的开始
3. 让sp-40，给当前栈帧分配空间

* ARM汇编函数的结束

```asm
  0x000104c8 <+84>: mov r0, r3
  0x000104cc <+88>: sub sp, r11, #4
  0x000104d0 <+92>: pop {r11, pc}
  0x000104d4 <+96>: andeq r0, r1, r12, asr #10
  0x000104d8 <+100>: andeq r0, r1, r0, asr r5
```

1. 把返回值存入r0
2. 让sp指向FP-4的位置
3. 依次把当前栈中的值弹出到pc和FP中，把进入函数时的LR填入PC，从而让处理器执行下一行指令

* 函数调用

```asm
0x000104bc <+72>: bl 0x10334 <getaddrinfo@plt>
```

`bl`是branch-and-link指令，跳转到新的函数地址，并把当前PC的值存入LR寄存器作为返回地址。

**plt** Procedure Linkage Table，库加载的函数，参见

https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html

继续执行程序到main函数返回前的17行后，在查看当前栈的最大深度

```bash
(gdb) scan_stack 0 $stack_size
Scanned 10000
Scanned 20000
Scanned 30000
Scanned 40000
Scanned 50000
Scanned 60000
Scanned 70000
Scanned 80000
Scanned 90000
Scanned 100000
Scanned 110000
Found data 11648 bytes deeper than current stack frame (0x7effeeb0).
Address    2130690352 = 0x7effc130
Stack size   135168 = 0x21000 = 132.0KB, 0x7efdf000-0x7f000000
Stack offset 119088 = 0x1d130 = 116.3KB
Stack depth   16080 = 0x03ed0 =  15.7KB
0x7effc130: 0x76ff94b0 0x7effc1a8 0x76e66c28 0x000004b0
0x7effc140: 0x7effc1ac 0x76fd8548 0x00000001 0x76e6c754
0x7effc150: 0x000004b0 0x76e70804 0x76ff94b0 0x7effc1ac
0x7effc160: 0x7effc1a8 0x00000000 0x76ffecf0 0x76e70804
```

此时的最大深度变为了15.7KB，说明执行过程某一个函数栈顶指向到了`0x7effc130`的位置

重启程序，并在执行到在main函数的断点后，增加一个**数据断点**，当指定的地址值发生变化时，触发断点

`(gdb) watch *(int*)0x7effc130`

继续执行`(gdb) c`后，程序断点在

```bash
Hardware watchpoint 3: *(int*)0x7effc130

Old value = 0
New value = 1996461232
check_match (undef_name=undef_name@entry=0x76df8116 "strcasecmp", ref=0x76df775c, ref@entry=0x59c2869, version=0x22e80, version@entry=0x76fffabc, flags=1, flags@entry=2, type_class=type_class@entry=1, sym=0x76e6c754, 
    sym@entry=0x770037f0, symidx=symidx@entry=1200, strtab=0x76e70804 "", strtab@entry=0x0, map=map@entry=0x76ff94b0, versioned_sym=versioned_sym@entry=0x7effc1ac, num_versions=num_versions@entry=0x7effc1a8) at dl-lookup.c:92
92 dl-lookup.c: No such file or directory.
```

说明执行到这个`check_match`函数时，栈深度增加到了最大值。此时需要分析包括这个函数在内的所有函数的栈帧空间大小。

```bash
(gdb) set height 0
(gdb) stack_walk
#0  check_match (undef_name=undef_name@entry=0x76df8116 "strcasecmp", ref=0x76df775c, ref@entry=0x59c2869, version=0x22e80, version@entry=0x76fffabc, flags=1, flags@entry=2, type_class=type_class@entry=1, sym=0x76e6c754, 
    sym@entry=0x770037f0, symidx=symidx@entry=1200, strtab=0x76e70804 "", strtab@entry=0x0, map=map@entry=0x76ff94b0, versioned_sym=versioned_sym@entry=0x7effc1ac, num_versions=num_versions@entry=0x7effc1a8) at dl-lookup.c:92
92 in dl-lookup.c
Top stack frame 0x7effc130
.......
#13 0x76e1e340 in _nss_dns_gethostbyname4_r (name=name@entry=0x10550 "test.example.com", pat=pat@entry=0x7effe998, buffer=0x7effea88 "\177", buflen=1024, errnop=errnop@entry=0x7effe99c, herrnop=herrnop@entry=0x7effe9ac, 
    ttlp=ttlp@entry=0x0) at nss_dns/dns-host.c:326
326 nss_dns/dns-host.c: No such file or directory.
Last stack frame 0x7effdbe0, current 0x7effe068, size of last 1160 = 0x488, total deeper   7992 = 0x01f38 =   7.8KB

#14 0x76f1dee0 in gaih_inet (name=<optimized out>, name@entry=0x10550 "test.example.com", service=<optimized out>, req=0x7effeeb4, pai=pai@entry=0x7effea40, naddrs=<optimized out>, naddrs@entry=0x7effea4c, 
    tmpbuf=<optimized out>, tmpbuf@entry=0x7effea80) at ../sysdeps/posix/getaddrinfo.c:848
848 ../sysdeps/posix/getaddrinfo.c: No such file or directory.
Last stack frame 0x7effe068, current 0x7effe8e0, size of last 2168 = 0x878, total deeper  10160 = 0x027b0 =   9.9KB

#15 0x76f1f010 in __GI_getaddrinfo (name=<optimized out>, service=<optimized out>, hints=<optimized out>, pai=0x7effeeb0) at ../sysdeps/posix/getaddrinfo.c:2391
2391 in ../sysdeps/posix/getaddrinfo.c
Last stack frame 0x7effe8e0, current 0x7effe9e8, size of last  264 = 0x108, total deeper  10424 = 0x028b8 =  10.2KB

#16 0x000104c0 in main () at test-getaddrinfo.c:16
16     int result = getaddrinfo("test.example.com", "80", &hints, &address_list);
Last stack frame 0x7effe9e8, current 0x7effeeb0, size of last 1224 = 0x4c8, total deeper  11648 = 0x02d80 =  11.4KB
```

由于这个`stack_walk`函数每次输出的是上一个函数的栈帧大小，所以frame 16的size of last 1224说明了frame15的大小为1224字节。切换到frame 15，查看这个函数具体做了什么

```asm
(gdb) f 15
#15 0x76f1f010 in __GI_getaddrinfo (name=<optimized out>, service=<optimized out>, hints=<optimized out>, pai=0x7effeec0) at ../sysdeps/posix/getaddrinfo.c:2391
2391 ../sysdeps/posix/getaddrinfo.c: No such file or directory.

(gdb) disassemble 
Dump of assembler code for function __GI_getaddrinfo:
   0x76f1eef0 <+0>: push {r4, r5, r6, r7, r8, r9, r10, r11, lr}
   0x76f1eef4 <+4>: add r11, sp, #32
   0x76f1eef8 <+8>: ldr r6, [pc, #2712] ; 0x76f1f998 <__GI_getaddrinfo+2728>
   0x76f1eefc <+12>: sub sp, sp, #1184 ; 0x4a0
```

可以看到这个函数在开始时分配了1184字节的栈空间`sub sp, sp, #1184`

从 https://code.woboq.org/userspace/glibc/sysdeps/posix/getaddrinfo.c.html 找到源代码

感觉frame14知道这个函数接下来调用的是`gaih_inet`，而这个函数在2265行，说明代码已经有了一些差异了，不过不影响。

```c
2263       struct scratch_buffer tmpbuf;
2264       scratch_buffer_init (&tmpbuf);
2265       last_i = gaih_inet (name, pservice, hints, end, &naddrs, &tmpbuf);
```

在这个函数之前有个结构体buffer，从名字上看就是要占用很大空间。转到这个结构体的定义

```c
struct scratch_buffer {
  void *data;    /* Pointer to the beginning of the scratch area.  */
  size_t length; /* Allocated space at the data pointer, in bytes.  */
  union { max_align_t __align; char __c[1024]; } __space;
};
```

还真有1024字节的数组buffer。

Frame 14的输出记录了Frame 13占用了2168的栈空间

```asm
(gdb) f 13
#13 0x76e1e340 in _nss_dns_gethostbyname4_r (name=name@entry=0x10550 "test.example.com", pat=pat@entry=0x7effe9a8, buffer=0x7effea98 "\177", buflen=1024, 
    errnop=errnop@entry=0x7effe9ac, herrnop=herrnop@entry=0x7effe9bc, ttlp=ttlp@entry=0x0) at nss_dns/dns-host.c:326
326 nss_dns/dns-host.c: No such file or directory.

(gdb) disassemble 
Dump of assembler code for function _nss_dns_gethostbyname4_r:
   0x76e1e268 <+0>: push {r4, r5, r6, r7, r8, r9, r10, r11, lr}
   0x76e1e26c <+4>: add r11, sp, #32
   0x76e1e270 <+8>: ldr r4, [pc, #812] ; 0x76e1e5a4 <_nss_dns_gethostbyname4_r+828>
   0x76e1e274 <+12>: sub sp, sp, #76 ; 0x4c
```

但是看函数栈初始化只是增加了76字节，没有2000多啊 ，通过查看`_nss_dns_gethostbyname4_r`的函数实现，其中有一句

```c
host_buffer.buf = orig_host_buffer = (querybuf *) alloca (2048);
```

根据Linux手册描述`alloca`函数分配栈上的空间 https://linux.die.net/man/3/alloca

> The **alloca**() function allocates *size* bytes of space in the stack frame of the caller. This temporary space is automatically freed when the function that called **alloca**() returns to its caller.

剩下的几个函数中都使用了`char tname[MAXDNAME+1]`这样的buffer来存储最大域名，但是每一个函数都有一份这个buffer，导致累加起来中共就有11K了。

 所以，对于嵌入式的平台，一般有特定的库，而不是通用的Linux库，不然栈都不够用的。



#### GDB工具脚本

作者写了几个函数用来查看函数的栈帧大小，以及栈空间的深度，即运行过程中栈顶的最大值

https://sourceware.org/gdb/onlinedocs/gdb/Define.html#index-user_002ddefined-command

```bash
# Functions for examining and manipulating the stack in gdb.

# Script constants.
set $one_kb        = 1024.0
set $safety_margin = 16

# Raspbian Linux stack parameters.
set $stack_start   = 0x7efdf000
set $stack_end     = 0x7f000000
set $stack_size    = $stack_end - $stack_start

define stack_args
    if $argc < 2
        printf "Usage: stack_args <offset|start> <length|end>\n"
    else
        if $arg0 < $stack_start
            # Assume arg0 is a relative offset from start of stack.
            set $offset = (int)$arg0
        else
            # Assume arg0 is an absolute address, so compute its offset.
            set $offset = (int)$arg0 - $stack_start
        end
        
        if $arg1 < $stack_start
            # Assume arg1 is a relative length.
            set $length = (int)$arg1
        else
            # Assume arg1 is an absolute address, so compute its length.
            set $length = (int)$arg1 - $stack_start - $offset
        end
    end
end

document stack_args
Usage: stack_args <offset|start> <length|end>

Set stack region offset and length from arguments.
end

define dump_stack
    if $argc < 2
        printf "Usage: dump_stack <offset|start> <length|end>\n"
    else
        stack_args $arg0 $arg1
        
        set $i = 0
        while $i < $length
            set $addr = $stack_start + $offset + $i
            x/4wx $addr
            set $i = $i + 16
        end
    end
end

document dump_stack
Usage: dump_stack <offset|start> <length|end>

Dumps stack starting at <offset|start> bytes, 4 longwords at a time,
for <length|end> bytes.
end

define clear_stack
    if $argc < 2
        printf "Usage: clear_stack <offset|start> <length|end>\n"
    else
        stack_args $arg0 $arg1
        
        if $stack_start + $offset + $safety_margin >= $sp
            printf "Error: start is in active stack.\n"
        else
            if $stack_start + $offset + $length + safety_margin >= $sp
                printf "Error: end is in active stack.\n"
            else
                set $i = 0
                while $i < $length
                    set $addr = $stack_start + $offset + $i
                    set *((int *) $addr) = 0
                    set $i = $i + 4
                    
                    # Takes a while, so give some feedback.
                    if $i % 10000 == 0
                        printf "Cleared %d\n", $i
                    end
                end
            end
        end
    end
end

document clear_stack
Usage: clear_stack <offset|start> <length|end>

Clears stack starting at <offset|start> bytes, one longword at a time,
for <length|end> bytes.
end

define stack_offset
    if $argc < 1
        printf "Usage: stack_offset <address>\n"
    else
        # Cast to int is needed to set $depth when $arg0 is $sp.
        set $addr   = (int)$arg0
        set $offset = $addr - $stack_start
        set $depth  = $stack_end - $addr
        
        printf "Address    %10d = 0x%08x\n", $addr, $addr
        
        if $addr < $stack_start || $addr >= $stack_end
            printf "Warning: address is not in stack.\n"
        end
        
        printf "Stack size   %6d = 0x%05x = %5.1fKB, 0x%x-0x%x\n", $stack_size, $stack_size, $stack_size / $one_kb, $stack_start, $stack_end
        printf "Stack offset %6d = 0x%05x = %5.1fKB\n", $offset, $offset, $offset / $one_kb
        printf "Stack depth  %6d = 0x%05x = %5.1fKB\n", $depth, $depth, $depth / $one_kb
    end
end

document stack_offset
Usage: stack_offset <address>

Shows stack offset and depth represented by address.
end

define scan_stack
    if $argc < 2
        printf "Usage: scan_stack <offset|start> <length|end>\n"
    else
        stack_args $arg0 $arg1
        
        set $addr = $stack_start + $offset
        set $i    = 0
        while $i < $length && *((int *) $addr) == 0
            set $addr = $stack_start + $offset + $i
            set $i = $i + 4
            
            # Takes a while, so give some feedback.
            if $i % 10000 == 0
                printf "Scanned %d\n", $i
            end
        end

        if *((int *) $addr) != 0
            if $addr < $sp
                set $offset = $sp - $addr
                printf "Found data %d bytes deeper than current stack frame (0x%x).\n", $offset, $sp
            else
                printf "Stack is clear up to current stack frame (0x%x), it is deepest stack usage.\n", $sp
            end
            
            stack_offset $addr
            dump_stack $addr-$stack_start 64
        else
            printf "Stack is clear in requested range.\n"
        end
    end
end

document scan_stack
Usage: scan_stack <offset|start> <length|end>

Scans stack for non-zero contents starting at <offset|start> bytes, one
longword at a time, for <length|end> bytes.
end

define stack_walk
    set $first_sp = $sp
    set $last_sp  = $sp
    set $total    = 0
    frame
    printf "Top stack frame 0x%08x\n\n", $last_sp
    
    # Loop will error out gracefully when there are no more frames.
    while 1
        up
        set $delta   = $sp - $last_sp
        set $total   = $total + $delta
        printf "Last stack frame 0x%08x, current 0x%08x, size of last %4d = 0x%03x, total deeper %6d = 0x%05x = %5.1fKB\n\n", $last_sp, $sp, $delta, $delta, $total, $total, $total / $one_kb
        set $last_sp = $sp
    end
end

document stack_walk
Usage: stack_walk

Walks stack frames upward from currently selected frame and computes
incremental and cumulative size of frames, so that stack consumption
can be attributed to specific functions.

Use "f 0" to select deepest frame of call stack, or "f <n>" to select
frame <n> higher up in stack.
end
```

