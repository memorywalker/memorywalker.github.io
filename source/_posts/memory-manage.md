---
title: 内存管理
date: 2020-03-06 20:25:49
categories:
- program
tags:
- memory
- linux
- malloc
---



## 内存


虚拟内存管理的最小单位为**页**，一个页可以是4K或8K

**段**是一个进程的数据或代码的逻辑分组，段不是连续的

现在的操作系统同时使用段和页，一个进程被分为多个段，每个段又有页

对于内存块的分配算法，不同的应用场景效率是不一样的。

#### Buddy memory allocation

https://en.wikipedia.org/wiki/Buddy_memory_allocation  

把内存分割为小块，尽可能的满足内存的分配需求。1963年Harry Markowitz发明

buddy分配方案有多种实现策略，最简单的是2分法。每一个内存块都有一个编号(order)，这个编号从0开始到n，编号为n的内存块的大小为`2**n`。当一个大的块被分割为两个相同的小块时，这两个小块就是buddy。只有两个buddy才能合并为一个大块。

一个块的最小大小值为2的0次方，即order为0的大小。

需要分配的内存大小为s，分配的块的order为x，则需要满足 `2**(x-1)<s<2**(x)`,即s大于order为x的大小的一半。

oder的最大值由系统可用的内存大小和最小块大小决定。例如最小块大小即order-0的大小为4K，对于一个有2000K内存的系统，order的最大值为8.因为对于order-8这个块，他的大小为2的8次方256*块的最小值4K为1024K，大于2000的一半了，所以如果order为9，就会超过2000的总大小。

###### 举例：

一个系统中的最小块大大小为64K，order的最大值为4，系统一次可以分配的内存大小最大值为`(2**4)*64=1024K`.假定系统的内存刚好也就1024K大小。

![buddyexp](/uploads/memory/buddyexp.png)

1. 初始状态
2. 程序A需要34K内存，因此order-0的块分配给A用就足够了，因为最小就是64.但是当前系统没有0的块，只有一个order是4的块，所以这个为4的块就一次一次对半分割，直到得到一个order-0，并把最左侧的给A使用。分割的过程中会产生一些其他块，这些块以free-list进行管理起来
3. 程序B需要66K内存，需要把order-1的块给B用，从当前的链表中发现已经有对应大小的块了，所以把对于的块之间给B用
4. 程序C需要35K内存，需要一个order-0的块给C用，现在刚好还有
5. 程序D需要67K内存，需要一个order-1的块，而此时没有order-1的块了，那就把order-2的块分解为两个order-1的块，把其中一个给D
6. 程序B释放了资源，此时order-1就多了一块出来，但是他不能和另一个order-1进行合并，因为他们不是来自同一个块，不是buddy
7. 程序D释放了资源，此时又一个order-1空出来了，发现他有buddy，所以他们可以合并为order-2

Buddy方案会导致内存浪费internal fragmentation，例如66K的内存需要order-1，其中近一半都被浪费了。

Linux内核使用buddy时进行了改进，同时结合了其他分配方案来管理内存块。

#### Slab Allocation

### 进程内存分段

一个进程使用的内存分为以下几个段

代码段(Text) ：存放可执行文件的指令即代码，只读避免程序被修改

数据段：存储可执行文件中已经初始化好的全局变量，静态分配的变量和全局变量

BSS：程序中未初始化的全局变量，值全部为0，内存位置连续

堆：动态分配的内存段，连续的内存，malloc使用，地址向大扩展

栈：程序执行中的局部变量，函数参数，返回值，地址向小扩展



brk, sbrk可以修改**program break**的位置，即heap的大小。

**sbrk()** increments the program's data space by *increment* bytes. 成功返回上一次的**program break**的位置。因此`sbrk((ptrdiff_t)0) `就可以返回当前的**program break**.

**brk()** sets the end of the data segment to the value specified by *addr*。成功返回0，这里的data segment并不是数据段。


http://man7.org/linux/man-pages/man2/sbrk.2.html

![linuxmemory](/uploads/memory/linuxmemory.png)

进程地址空间分为用户空间和内核空间。用户空间从0到0xC0000000，内核空间使用剩下的高地址部分。用户进程只有进行系统调用才可以访问内核空间。每个进程使用自己的用户空间，而内核空间是内核负责，不会随着进程改变而变化。内核空间地址有自己对应的页表。用户进程各自有不同的页表。

逻辑地址经过段机制转化为线性地址，线性地址经过页机制转化为物理地址

使用`cat /proc/<pid>/maps`查看进程的内存区域

内核使用`vm_area_struct`描述进程地址空间的基本管理单元，使用链表进行链接这些块，以红黑树的形式组织。遍历时使用链表，定位内存位置时使用红黑树

内核使用`do_mmap()`函数创建一个新的线性地址空间

### 参考资料

* xxx











