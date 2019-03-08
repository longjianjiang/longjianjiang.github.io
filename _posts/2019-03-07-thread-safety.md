---
layout: post
title:  "多线程学习笔记"
date:   2019-03-07
excerpt:  "本文是笔者学习多线程的笔记"
tag:
- OS
comments: true
---

> 多线程问题不可否认是一个复杂的问题，本文笔者将自己对多线程的学习和理解记录下来。

首先我们想一下阻塞队列和并发队列如何实现？其实我们需要解决的问题就是生产者和消费者的问题，我们需要达到的目的就是不同个数的生产者和消费者可以正常的操作物品，生产者不会在存储满时继续生产，消费者也不会在存储为空时继续消费。

上述所说的也就是所谓的多线程同步问题。

阻塞队列我们可以通过`mutex`和`conditoin_variable`实现同步，实现方式是通过阻塞线程。

并发队列则不阻塞线程的方式达到线程的同步。通过原子操作，也就是CAS（Compare And Swap）。

[ref1](https://blog.poxiao.me/p/spinlock-implementation-in-cpp11/)
[ref2](https://www.miaoerduo.com/c/c-%E5%B9%B6%E5%8F%91%E9%98%9F%E5%88%97%E7%9A%84%E5%8E%9F%E7%90%86%E7%AE%80%E4%BB%8B%E4%B8%8E%E5%BC%80%E6%BA%90%E5%BA%93concurrentqueue%E5%AE%89%E5%88%A9.html)
[ref3](https://www.ibm.com/developerworks/cn/linux/l-cn-lockfree/index.html)