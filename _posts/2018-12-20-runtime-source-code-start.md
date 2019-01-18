---
layout: post
title:  "【Runtime源码】前言"
date:   2018-12-20
excerpt:  "本文是笔者阅读Runtime源码的自我计划"
tag:
- Runtime
comments: true
---


## 开始

之前写过一篇关于Runtime的文章，不过只是简单介绍了Runtime是什么，并没有去了解他的实现。

都说阅读源码很有帮助，所以笔者在之前的一段时间也阅读过一些开源库的源代码，笔者阅读的第一个开源库是 fishhook，因为fishhook比较简单，所以阅读起来也没有那么耗时，但是阅读完以后，笔者认为给了自己一种自信。

Runtime的源码就比较多了，而且关于Runtime源码的文章也很多，但是看别人的不如自己体验一遍并写成文印象来的深刻，所以就有了笔者的【Runtime源码】系列文章，就当是记录了自己阅读的过程。

> 可以从[这里]下载一份可以编译的Runtime源码。

## C函数

因为Runtime用C++实现，所以不可避免的会用到一些C函数，笔者在这里记录下来。

`void *memcpy(void *__dst, const void *__src, size_t __n);`
将src地址开始的连续n个字节数据复制到以dst地址开始的空间内。

`void * memmove(void *dest, const void *src, size_t num);`
和 memcpy 一样的功能，不过当src 和 dest 内存有重叠时，memmove 还是能正确处理，方法内部应该有先复制src内容，然后在move。

`void* malloc (size_t size);`
动态内存分配，向系统请求在堆中分配 size 大小的内存的空间。

`void* calloc (size_t num, size_t size);`
类似 malloc，向系统请求分配 num 个 大小为 size 的内存空间，不过分配内存空间的同时，同时将这段内存空间的每一个字节初始化为0。

`void* realloc (void* ptr, size_t size);`
重新为地址 ptr 分配 size 大小的空间。当使用 malloc 和 calloc 分配的内存空间不够使用，可以使用该方法进行调整。

`int strcmp(const char *s1, const char *s2);`
比较两个字符串是否相等，当返回值为0说明字符串相等。

## 然后

笔者记录过程如下:

- 类的结构(isa)
- 类的结构(class_data_bits_t)
- 类的加载过程
- 类的propety 和 ivar
- 类的方法及调用过程
- 类的load方法 和 initialize方法
- 类的分类

## 最后

通过阅读Runtime的源码，笔者总结了一些自己学到的东西。

- 判断的代码必不可少，Runtime源码中可以发现很多方法都有加断言，这样可以避免问题的发生和问题的定位。



