---
layout: post
title:  "【Runtime源码】autorelease"
date:   2019-04-16
excerpt:  "本文是笔者阅读Runtime源码关于自动释放池实现的笔记"
tag:
- Runtime
comments: true
---

> 本文笔者根据 Runtime 750 源码分析自动释放池的实现。

之前笔者在[内存管理](http://www.longjianjiang.com/memory-management/)中提到过autorelease，本文来介绍Runtime中如何实现这个机制的。

## 开始

所谓autorelease，就是不用手动去写release方法，一个所谓的Pool的最后会帮我们调用release。下面来看一个例子:

{% highlight objc %}
id objc_autorelease(id obj) {
    if (!obj) return obj;
    if (obj->isTaggedPointer()) return obj;
    return obj->autorelease();
}

id objc_retainAutorelease(id obj) {
    return objc_autorelease(objc_retain(obj));
}

int main() {
    @autoreleasepool {
        id obj = [Sark new];
        id __autoreleasing ar_obj_1 = [Sark new];
        id __autoreleasing ar_obj_2 = obj;
    }
}
{% endhighlight %}

ar_obj_1赋值后，会调用一个`objc_autorelease`，也就是将`arr_obj_1`加入了Pool中；而arr_obj_2则会收到obj调用`objc_retainAutorelease`后的返回值。

ARC中只有将对象赋值给一个`__autoreleasing`修饰的左值，才会将其加入Pool中，否则是局部变量超过其作用域，从而导致其指向的对象释放。

## 数据机构

前面main函数中的`@autoreleasepool{}`，实际会被编译器转成如下代码:

{% highlight cpp %}
void *context = objc_autoreleasePoolPush();
// {}中的代码
objc_autoreleasePoolPop(context);
{% endhighlight %}

继续看下这两个方法的实现：

{% highlight cpp %}
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
{% endhighlight %}

可以看到实际的实现是在`AutoreleasePoolPage`这个类。

{% highlight cpp %}
class AutoreleasePoolPage {
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)

#   define POOL_BOUNDARY nil
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;

    static void * operator new(size_t size) {
        return malloc_zone_memalign(malloc_default_zone(), SIZE, SIZE);
    }
    id * begin() {
        return (id *) ((uint8_t *)this+sizeof(*this));
    }

    id * end() {
        return (id *) ((uint8_t *)this+SIZE);
    }

    id *add(id obj) {
        assert(!full());
        unprotect();
        id *ret = next;  // faster than `return next-1` because of aliasing
        *next++ = obj;
        protect();
        return ret;
    }
};
{% endhighlight %}

实际autorelease的对象就是存储在这个PoolPage构成的一个双向链表中。通过重载了`new`，每一个PoolPage会分配一页虚拟内存页的大小。

`begin()`指向实际开始存储对象的位置，`end()`则指向一块内存的最后，插入的时候方便进行判断。

`add(id)`方法则是将对象加入到Pool中，并且返回的ret指向的值就是obj。可以看到next指向的就是当前Pool的空闲位置。

`EMPTY_POOL_PLACEHOLDER`是一个空的占位的Pool，存储在TLS(Thread Local Storage)中，注释中说主要是为了节省内存，因为可能存在只创建但是从不使用Pool。

所以现在我们大概知道了这个类类似一个栈，存储了autorelease的对象，next总是指向栈顶。

## Push & Pop

首先来看`AutoreleasePoolPage`中Push的实现:

{% highlight cpp %}
static inline id *autoreleaseFast(id obj) {
    AutoreleasePoolPage *page = hotPage();
    if (page && !page->full()) {
        return page->add(obj);
    } else if (page) {
        return autoreleaseFullPage(obj, page);
    } else {
        return autoreleaseNoPage(obj);
    }
}
static inline void *push() {
    id *dest;
    dest = autoreleaseFast(POOL_BOUNDARY);
    assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
    return dest;
}
{% endhighlight %}


static inline void pop(void *token) {
    AutoreleasePoolPage *page;
    id *stop;

    page = pageForPointer(token);
    stop = (id *)token;
    if (*stop != POOL_BOUNDARY) {
        if (stop == page->begin()  &&  !page->parent) {
            // Start of coldest page may correctly not be POOL_BOUNDARY:
            // 1. top-level pool is popped, leaving the cold page in place
            // 2. an object is autoreleased with no pool
        } else {
            // Error. For bincompat purposes this is not 
            // fatal in executables built with old SDKs.
            return badPop(token);
        }
    }

    page->releaseUntil(stop);

    // memory: delete empty children
    if (DebugPoolAllocation  &&  page->empty()) {
        // special case: delete everything during page-per-pool debugging
        AutoreleasePoolPage *parent = page->parent;
        page->kill();
        setHotPage(parent);
    } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
        // special case: delete everything for pop(top) 
        // when debugging missing autorelease pools
        page->kill();
        setHotPage(nil);
    } 
    else if (page->child) {
        // hysteresis: keep one empty child if page is more than half full
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
{% endhighlight %}
## add

## release

## 总结
