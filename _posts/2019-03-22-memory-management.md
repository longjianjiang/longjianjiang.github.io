---
layout: post
title:  "内存管理学习笔记"
date:   2019-03-22
excerpt:  "本文是笔者学习内存管理的笔记"
tag:
- OS
comments: true
---

> 内存管理是一个很基础的问题，本文笔者记录下自己的总结过程。本文中所讨论的内存是进程运行时动态分配的，这一部分内存存放在进程的堆段。

不论我们如何对内存进行管理，我们都得向操作系统申请内存，所以首先需要了解操作系统是如何进行管理内存的。这部分内容可以参考[这里](http://www.longjianjiang.com/os-memory-management/)。

## malloc & free

> 这里不会深入malloc和free的具体实现。

`malloc` 方法笔者第一次接触是在大学C语言中，知道这个方法是向操作系统申请n个字节的内存，并返回这一段连续内存的首地址，如果失败则返回NULL。

有使用对应的就会有释放，`free` 就是来释放之前向操作系统申请的那一段内存。

现在让我们想一个问题，`free` 方法仅接收一个首地址，它是如何知道释放多长的内存的？

其实这个解决方案不唯一，有的malloc的实现内部将分配的内存size进行分组，根据传入的size返回合适的size的内存段。

还有一种实现则是增加一个结构体用来存储分配的size，所以malloc方法返回的地址其实就是结构体的地址往前偏移了结构体size的一个新的地址，所以free的时候只需要将这个地址往后编译结构体size就得到了结构体的地址，通过结构体地址自然也就知道了当初申请内存的大小。

其实不光有malloc，还有 `realloc` 这个方法的作用是当 `malloc` 分配的内存不够使用时，使用`realloc`可以继续申请连续size个内存空间，同时每一个字节初始化为0。

## 引用计数

## 内存池

## 垃圾回收

## iOS中的内存管理

## References

[https://www.ibm.com/developerworks/linux/library/l-memory/](https://www.ibm.com/developerworks/linux/library/l-memory/)

[https://opensource.apple.com/source/libmalloc/libmalloc-53.1.1/src/malloc.c.auto.html](https://opensource.apple.com/source/libmalloc/libmalloc-53.1.1/src/malloc.c.auto.html)


