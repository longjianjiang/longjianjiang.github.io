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
struct __AtAutoreleasePool {
    __AtAutoreleasePool() {
        atautoreleasepoolobj = objc_autoreleasePoolPush();
    }
    ~__AtAutoreleasePool() {
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    void * atautoreleasepoolobj;
};
/* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool;
    ...
}
{% endhighlight %}

`__AtAutoreleasePool`初始化时，调用了`objc_autoreleasePoolPush`，当`__AtAutoreleasePool`超出作用域，调用析构函数，从而调用`objc_autoreleasePoolPop`，继续看下这两个方法的实现：

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
#   define POOL_BOUNDARY nil
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

Pool的`push`方法可以看到就是去调用了`autoreleaseFast`方法，同时传了一个`POOL_BOUNDARY`的参数，这个参数就是nil。

而`autoreleaseFast`方法又分了三种情况，不过除了之前说过的`EMPTY_POOL_PLACEHOLDER`之外，其他的都调用了Pool的add方法将obj也就是一个nil加入到Pool中，同时也返回了nil。

1.取hotPage，这个方法就是从TLS中根据key取到当前一个还没满的页，每当创建一个新的页的时候，会使用`setHotPage`进行更新；

2.1.hotPage不为空，此时直接调用add方法；

2.2.hotPage不为空，但满了，此时调用`autoreleaseFullPage`创建一个新的页，使用`setHotPage`进行更新，并在新的页中插入obj；

2.3.hotPage为空，第一次初始化一个页，调用`autoreleaseNoPage`，创建一个新的页，使用`setHotPage`进行更新，并在新的页中插入obj；这里插入obj之前，会判断是否存在`EMPTY_POOL_PLACEHOLDER`，如果存在的话，会额外插入一个nil到页中。

到这里，其实我们可以发现`push`方法的主要作用就是往hotPage中插入一个nil，因为`@autoreleasepool{}`可以嵌套，所以每进行一次push插入一个nil，就是为了区分当前是在那个嵌套中，这样删除的时候就有标记了。

下面继续来看pop的实现：

{% highlight cpp %}
void releaseUntil(id *stop) {
    while (this->next != stop) {
        AutoreleasePoolPage *page = hotPage();

        while (page->empty()) {
            page = page->parent;
            setHotPage(page);
        }

        page->unprotect();
        id obj = *--page->next;
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
        page->protect();

        if (obj != POOL_BOUNDARY) {
            objc_release(obj);
        }
    }

    setHotPage(this);
}

static inline void pop(void *token) {
    AutoreleasePoolPage *page;
    id *stop;

    if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
        if (hotPage()) {
            pop(coldPage()->begin());
        } else {
            setHotPage(nil);
        }
        return;
    }

    page = pageForPointer(token);
    stop = (id *)token;
    if (*stop != POOL_BOUNDARY) {
        if (stop == page->begin()  &&  !page->parent) {
        } else {
            return badPop(token);
        }
    }

    page->releaseUntil(stop);

    if (DebugPoolAllocation  &&  page->empty()) {
        AutoreleasePoolPage *parent = page->parent;
        page->kill();
        setHotPage(parent);
    } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
        page->kill();
        setHotPage(nil);
    } 
    else if (page->child) {
        if (page->lessThanHalfFull()) {
            page->child->kill();
        }
        else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
{% endhighlight %}

pop的参数其实就是push的返回值，而且只要两种可能，要么是nil要么是`EMPTY_POOL_PLACEHOLDER`。

1.首先判断token是否为`EMPTY_POOL_PLACEHOLDER`，此时如果发生了嵌套，hotPage就不为空，所以需要pop，此时pop的是`EMPTY_POOL_PLACEHOLDER`嵌套里的所有对象。否则将hotPage置为nil；

2.根据token，获得token所在的页，同时验证token一定是nil；

3.调用page的`releaseUntil`方法，该方法其实就是不断将next指针往下移动，取到对象调用`objc_release`，如果当前页空了，则根据parent指针去前一页继续操作，直到next指向了nil，也就是到了一个嵌套的开始，此时结束一个Pool的移除操作。

4.最后则尝试使用`kill`移除空的页；

PoolPage和线程是对应的，在PoolPage初始化的时候，设置了tls的一个销毁回调`tls_dealloc`，这个回调会定位到链表的首节点的页，然后调用pop，相当于移除最外层Pool。

到了这里，基本了解了PoolPage的工作原理。其实很简单，就是一个双链表存储了所有autorelease对象，同时每一个嵌套开始插入一个nil，这样清除的时候，清除到nil为止，一个嵌套清除结束。

## autorelease

autorelease的实际实现同样也在PoolPage中，如下所示：

{% highlight cpp %}
static inline id autorelease(id obj) {
    assert(obj);
    assert(!obj->isTaggedPointer());
    id *dest __unused = autoreleaseFast(obj);
    assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
    return obj;
}
{% endhighlight %}

可以看到很简单，就是调用了之前所说的`autoreleaseFast`，该方法再调用`add`将obj插入到page中即可。

## 总结

本文介绍了autorelease的实现，其实依然是一个数据结构，理解了这个数据结构就能知道其实现原理。