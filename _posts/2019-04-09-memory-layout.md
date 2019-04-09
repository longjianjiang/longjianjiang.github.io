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

### Stack

栈用来存放**stack frame**信息的，所谓栈帧其实就是栈内存上的一个片段，该片段就是一个函数调用，当函数返回时，栈帧对应的那段内存就会被回收。栈帧会为函数的局部变量分配内存，同时保存函数的返回地址。

当持续的重用栈空间时，CPU会将这部分内存放入缓存中从而加速访问。当函数调用过程中，由于调用层级过多，或者函数中不断为局部变量分配内存，当超过栈的最大容量时，会产生栈溢出，程序则收到一个段错误。

### MMAP

内存映射区用来实现内存和文件中的内容相互映射，可以使用`mmap`函数进行内存映射。[mmkv](http://www.longjianjiang.com/mmkv/)就是使用内存映射来进行持久化的一个key&value存储的库。

### Heap

堆主要用来存储运行时动态创建的对象。

### BSS & Data & Text

BSS(block started by symbol)段，存储了未初始化或初始化值为0的全局变量或者静态局部变量。

数据段，存储了初始化值不为0的全局变量或者静态局部变量。

代码段，存储了应用程序的CPU执行指令，也存储了字符串常量。这部分区域是可以被共享的，这也是之前所说的段式存储方便实现共享的特点。

### Reserved

保留段，位于地址的最低部分，没有对应的物理地址。

## References

[https://manybutfinite.com/post/anatomy-of-a-program-in-memory/](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)