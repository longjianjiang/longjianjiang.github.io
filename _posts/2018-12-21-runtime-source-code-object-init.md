---
layout: post
title:  "【Runtime源码】对象的创建和销毁"
date:   2018-12-21
excerpt:  "本文是笔者阅读Runtime源码关于对象创建的笔记"
tag:
- Runtime
comments: true
---

> 本文笔者根据 Runtime 750 源码分析OC中对象创建和销毁的流程。

下面是代码中会使用到的两个宏，笔者事先拿出来。

{% highlight cpp %}
#define fastpath(x) (__builtin_expect(bool(x), 1))
#define slowpath(x) (__builtin_expect(bool(x), 0))
{% endhighlight %}

`__builtin_expect(EXP, N)` 是一个指令，用来告诉编译器将最有可能执行的分支告诉编译器。 也就是 EXP 等于 N的概率比较大，这样编译器可以对分支转移进行优化，减少指令的跳转。

所以上面两个宏，slowpath 表示 表达式 x 等于 false 的概率比较大，fastpath 表示 表达式 x 等于 true 的概率比较大。

## 创建

OC 中我们创建对象基本上用 `[Person new]`, `[Person alloc] init` 两种方式来初始化一个对象，下面笔者看下Runtime源码中是如何一步一步为我们初始化一个对象的。

