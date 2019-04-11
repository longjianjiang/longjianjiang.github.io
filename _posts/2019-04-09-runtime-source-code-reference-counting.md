---
layout: post
title:  "【Runtime源码】引用计数"
date:   2019-04-09
excerpt:  "本文是笔者阅读Runtime源码关于引用计数的笔记"
tag:
- Runtime
comments: true
---

> 本文笔者根据 Runtime 750 源码分析引用计数的实现。

## taggedPointer

在64位系统中，为了节省内存和提高执行效率，苹果提出了taggedPointer，用来存储小对象，比如`NSNumber`,`NSDate`,`NSString`，taggedPointer的值其实并不是通过malloc函数返回的地址，而只是一个像地址的值实际存储的是真正的值和类型信息。

## retain & release

虽然现在是ARC，MRC时代可能绝大多数的iOS开发者并没有接触过，但是`retain`, `release`这两个操作引用计数的方法，一定都不会陌生。下面笔者根据runtime源码去看引用计数是如何存储，以及操作引用计数的实现。

首先我们先看`retain` 方法，既然是增加引用计数的方法，那么就一定会去取引用计数，所以也就知道了引用计数是如何进行存储的。

![runtime_source_code_rc_1]({{site.url}}/assets/images/blog/runtime_source_code_rc_1.jpg)

上面就是retain的一个调用栈，涉及的函数如下:

{% highlight objc %}
id objc_retain(id obj) {
    if (!obj) return obj;
    if (obj->isTaggedPointer()) return obj;
    return obj->retain();
}

id objc_object::retain() {
    assert(!isTaggedPointer());
    if (fastpath(!ISA()->hasCustomRR())) {
        return rootRetain();
    }
    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, SEL_retain);
}

- (id)retain {
    return ((id)self)->rootRetain();
}

id objc_object::rootRetain() {
    return rootRetain(false, false);
}

id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    ...
    return (id)this;
}
{% endhighlight %}


入口函数是`id objc_retain(id obj)`，可以看到函数首先判断对象是不是`taggedPointer`，如果是的话直接返回对象，因为taggedPointer的内存由栈管理，并不需要额外的内存管理。


