---
layout: post
title:  "内存布局学习笔记"
date:   2019-04-09
excerpt:  "本文是笔者学习内存布局的笔记"
tag:
- OS
comments: true
---

之前在[操作系统内存管理](http://www.longjianjiang.com/os-memory-management/)说到了一种段式存储的方案，本文主要介绍Linux进程具体如何进行分段，以及各分段的作用。

首先给出linux进程的分段图:

![memory_layout_1]({{site.url}}/assets/images/blog/memory_layout_1.png)

下面进行介绍各分段的作用:

### kernel space

内核总是驻留在内存中，因为要随时处理中断和系统调用。这部分内存应用程序没有权限进行读写。

当进程进行系统调用(system call) 时，系统调用依然是在当前进程的内存空间的内核区域中，这样系统调用的代码可以直接的访问到进程的用户区内存，也不需要切换页表。

### Stack

栈用来存放**stack frame**信息的，所谓栈帧其实就是栈内存上的一个片段，该片段就是一个函数调用，当函数返回时，栈帧对应的那段内存就会被回收。栈帧会为函数的局部变量分配内存，同时保存函数的返回地址。

当持续的重用栈空间时，CPU会将这部分内存放入缓存中从而加速访问。当函数调用过程中，由于调用层级过多，或者函数中不断为局部变量分配内存，当超过栈的最大容量时，会产生栈溢出，程序则收到一个段错误。

### MMAP

内存映射区用来实现内存和文件中的内容相互映射，可以使用`mmap`函数进行内存映射。[mmkv](http://www.longjianjiang.com/mmkv/)就是使用内存映射来进行持久化的一个key&value存储的库。

Q: mmap的实现？

mmap的实现分成几个阶段。

第一阶段是根据映射的范围找到一块空闲的虚拟地址空间，初始化一个vma（vm_area_struct），插入到进程的虚拟地址区域链表；调用内核空间的系统调用函数mmap（不同于用户空间函数），实现文件物理地址和进程虚拟地址的一一映射关系;

第二阶段是，当实际触发对虚拟地址空间进行访问的时候，发现数据不存在于物理内存空间时，触发Page Fault，Linux才会将缺失的Page换入内存空间；

mmap的优点其实在于将一块虚拟内存地址和文件page cache的映射，免去了常规read从page cache拷贝到用户空间的一次拷贝；

Q: 什么时候使用mmap？


---

malloc的实现中，如果请求的内存长度大于一定长度（128k）此时就会使用mmap的内存区域进行分配，正常在堆区域分配的内存，是通过压栈的形式进行分配。 

释放的时候需要等到高地址内存释放以后才能释放（例如，在B释放之前，A是不可能释放的，这就是内存碎片产生的原因，什么时候紧缩看下面），而mmap分配的内存可以单独释放。

当最高地址空间的空闲内存超过128K（可由M_TRIM_THRESHOLD选项调节）时，执行内存紧缩操作（trim）。在上一个步骤free的时候，发现最高地址空闲内存超过128K，于是内存紧缩。

[ref](https://www.cnblogs.com/vinozly/p/5489138.html)

### Heap

堆主要用来存储运行时动态创建的对象。

### BSS & Data & Text

BSS(block started by symbol)段，存储了未初始化或初始化值为0的全局变量或者静态局部变量。

数据段，存储了初始化值不为0的全局变量或者静态局部变量。

代码段，存储了应用程序的CPU执行指令，也存储了字符串常量。这部分区域是可以被共享的，这也是之前所说的段式存储方便实现共享的特点。

### Reserved

保留段，位于地址的最低部分，没有对应的物理地址。

## References

[https://github.com/mit-pdos/xv6-public](https://github.com/mit-pdos/xv6-public)

[https://manybutfinite.com/post/anatomy-of-a-program-in-memory/](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)
