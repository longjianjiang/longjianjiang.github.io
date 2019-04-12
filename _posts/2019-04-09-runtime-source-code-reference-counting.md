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

在64位系统中，为了节省内存和提高执行效率，苹果提出了`taggedPointer`，用来存储小对象，比如`NSString`，`taggedPointer`的值其实并不是通过malloc函数返回的地址，而只是一个像地址的值实际存储的是真正的值和类型信息。

一般我们创建的NSString实际类型是`NSTaggedPointerString`，很直观的名字，一看就知道此时返回的是`taggedPointer`。

## 开始

虽然现在是ARC，MRC时代可能绝大多数的iOS开发者并没有接触过，但是`retain`, `release`这两个操作引用计数的方法，一定都不会陌生。下面笔者根据runtime源码去看引用计数是如何存储，以及操作引用计数的实现。

## retain

首先我们先看`retain` 方法，既然是增加引用计数的方法，那么就一定会去取引用计数，所以也就知道了引用计数是如何进行存储的。

![runtime_source_code_rc_1]({{site.url}}/assets/images/blog/runtime_source_code_rc_1.png)

上面就是retain的一个调用栈，涉及的函数如下:

{% highlight cpp %}
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

id objc_object::rootRetain() {
    return rootRetain(false, false);
}

id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    ...
    return (id)this;
}
{% endhighlight %}

> 上述截取的是`SUPPORT_NONPOINTER_ISA`为true时的方法，关于`nonpointer`可以参考[这里](http://www.longjianjiang.com/runtime-source-code-class-isa/)。

入口函数是`id objc_retain(id obj)`，可以看到函数首先判断对象是不是`taggedPointer`，如果是的话直接返回对象，因为taggedPointer的内存由栈管理，并不需要额外的内存管理。

接下来就去到了`objc_object`的`retain()` 方法，该方法首先确认不是`taggedPointer`，接着判断了是否该类是否实现类自定义的retain/release方法，如果有的话则使用了`objc_msgSend`向对象发送retain消息，也就是去调用自定义的retain方法。

大多数情况下并不会实现自定义的retain/release方法，从使用[`fastpath`](http://www.longjianjiang.com/runtime-source-code-object-init/)也能看出来，所以会直接调用`objc_object`的`rootRetain()` 方法，而`rootRetain()` 方法只是简单的调用了另一个重载方法，具体的实现也在重载方法中。

之前在[isa](http://www.longjianjiang.com/runtime-source-code-class-isa/)中说到isa结构，截取引用计数相关的两位:

```
uintptr_t has_sidetable_rc  : 1; // 如果👇存储的引用计数大于10，该字段存储进位；
uintptr_t extra_rc          : 8;  // 存储对象的引用计数，当对象引用计数为10，该字段存9；
```

`extra_rc`只占了8位也就是最多存储引用计数到255，超过就会溢出，下面我们分情况来看`rootRetain(bool,bool)`的实现。

### 未溢出

{% highlight cpp %}
#define RC_ONE   (1ULL<<56)
uintptr_t LoadExclusive(uintptr_t *src) {
    return *src;
}

id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    if (isTaggedPointer()) return (id)this;

    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));

    return (id)this;
}
{% endhighlight %}

这个版本实现很直观，做了下面4步:

0.判断是否为`taggedPointer`；

1.获取isa的值，赋值给newisa；

2.更新newisa，将引用计数加1，因为isa结构中`extra_rc`存储在57位起始的8位，`RC_ONE`就是57位为1，这样bits和`RC_ONE`相加等于就是给引用计数加1了；

3.CAS原子操作使用newisa的bits更新isa，赋值没成功就重试，直到成功；

### 溢出

此时说明`extra_rc`的值加1后超过了255，此时就会溢出，下面来看溢出的实现:

{% highlight cpp %}
id objc_object::rootRetain_overflow(bool tryRetain) {
    return rootRetain(tryRetain, true);
}

id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++
        if (slowpath(carry)) {
            if (!handleOverflow) {
                ClearExclusive(&isa.bits);
                return rootRetain_overflow(tryRetain);
            }
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));

    if (slowpath(transcribeToSideTable)) {
        sidetable_addExtraRC_nolock(RC_HALF);
    }

    return (id)this;
}
{% endhighlight %}

使用`addc`进行引用计数加1时，会传一个carry的指针，如果carry被赋值，说明就溢出了，此时就需要处理溢出。

这里绕了一个弯子，因为最初调用`rootRetain(bool,bool)`第二个是否处理溢出传了false，所以当溢出时会去调用`rootRetain_overflow`方法，不过这个方法却还是调用了`rootRetain(bool,bool)`，只是将处理溢出的bool设置为了true。

所以继续之前的引用计数加1的操作，依然走到`if (slowpath(carry))`分支，此时才会去处理溢出，此时引用计数为256，根据注释，我们知道了将引用计数的一半继续留在`extra_rc`，另一半则存储到散列表中。同时也标记了isa中`has_sidetable_rc`为1。

接下来的工作就是将一半的引用计数存储到散列表中，具体实现在方法`sidetable_addExtraRC_nolock`中：

{% highlight cpp %}
// The order of these bits is important.
#define SIDE_TABLE_WEAKLY_REFERENCED (1UL<<0)
#define SIDE_TABLE_DEALLOCATING      (1UL<<1)  // MSB-ward of weak bit
#define SIDE_TABLE_RC_ONE            (1UL<<2)  // MSB-ward of deallocating bit
#define SIDE_TABLE_RC_PINNED         (1UL<<(WORD_BITS-1))

#define SIDE_TABLE_RC_SHIFT 2
#define SIDE_TABLE_FLAG_MASK (SIDE_TABLE_RC_ONE-1)

bool objc_object::sidetable_addExtraRC_nolock(size_t delta_rc) {
    assert(isa.nonpointer);
    SideTable& table = SideTables()[this];

    size_t& refcntStorage = table.refcnts[this];
    size_t oldRefcnt = refcntStorage;
    assert((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);
    assert((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);

    if (oldRefcnt & SIDE_TABLE_RC_PINNED) return true;

    uintptr_t carry;
    size_t newRefcnt = 
        addc(oldRefcnt, delta_rc << SIDE_TABLE_RC_SHIFT, 0, &carry);
    if (carry) {
        refcntStorage =
            SIDE_TABLE_RC_PINNED | (oldRefcnt & SIDE_TABLE_FLAG_MASK);
        return true;
    }
    else {
        refcntStorage = newRefcnt;
        return false;
    }
}
{% endhighlight %}

通过该方法的实现其实我们可以知道散列表中用来存储额外引用计数的低2位是用来标记析构和弱引用相关的，所以实际存储从第三位开始。

之后就是使用`addc`计算出当前引用计数和添加的引用计数的和，这里同样判断了是否溢出。

如果溢出，则将存储引用计数的值设置位`SIDE_TABLE_RC_PINNED`，(oldRefcnt & SIDE_TABLE_FLAG_MASK) 的值是0，因为前面断言了低2位为0。`SIDE_TABLE_RC_PINNED`是一个很大的值，所以之前有一个判断，如果当前引用计数等于`SIDE_TABLE_RC_PINNED`则直接返回了。

最后没有溢出，则使用前面相加的和更新散列表中额外存储的引用计数。

如果`extra_rc`第二次溢出，则继续将128存储到散列表中。不断重复，直到造成散列表存储溢出，后面则不进行处理了，不过一般情况达不到那么高的引用计数的。

## 引用计数的存储

根据retain方法的实现，我们也就得到了引用计数在OC中是如何进行存储的。通过retain方法我们知道了OC中引用计数(nonpointer为1)是存储在isa中8位，如果溢出则每次将一半也就是128额外存储在散列表中。同时当对象创建时引用计数为1，`extra_rc`为0，从名字也可以看出来，保存的其实是额外的引用计数。

## release

![runtime_source_code_rc_2]({{site.url}}/assets/images/blog/runtime_source_code_rc_2.png)

依然来看release的调用栈，涉及的函数如下:

{% highlight cpp %}
void objc_release(id obj) {
    if (!obj) return;
    if (obj->isTaggedPointer()) return;
    return obj->release();
}

void objc_object::release() {
    assert(!isTaggedPointer());

    if (fastpath(!ISA()->hasCustomRR())) {
        rootRelease();
        return;
    }

    ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_release);
}

bool objc_object::rootRelease() {
    return rootRelease(true, false);
}

bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow) {
}
{% endhighlight %}

可以看到调用结构和retain类似，具体将引用计数减1的在方法`rootRelease(bool, bool)`。同样的我们分情况来看该方法的实现:

### 未溢出

{% highlight cpp %}
bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow) {
    if (isTaggedPointer()) return false;

    isa_t oldisa;
    isa_t newisa;

    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
    } while (slowpath(!StoreReleaseExclusive(&isa.bits, 
                                             oldisa.bits, newisa.bits)));
    return false;
}
{% endhighlight %}

release未溢出实现和retain类似，做了下面4步:

0.判断是否为`taggedPointer`；

1.获取isa的值，赋值给newisa；

2.更新newisa，将引用计数减1，因为isa结构中`extra_rc`存储在57位起始的8位，`RC_ONE`就是57位为1，这样bits和`RC_ONE`相减等于就是给引用计数减1了；

3.CAS原子操作使用newisa的bits更新isa，赋值没成功就重试，直到成功；

因为release还有一个额外的操作，当引用计数为0时，需要释放内存，OC中需要调用dealloc方法。因为isa中`extra_rc`存储的是额外的引用计数，当`extra_rc`为0进行一次release后，如果没有额外的引用计数存储在散列表中，此时需要释放内存，否则则需要去散列表中去额外的引用计数，下面分别来看这两种情况。

### 释放内存

{% highlight cpp %}
bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow) {
    if (isTaggedPointer()) return false;

    isa_t oldisa;
    isa_t newisa;
retry:
    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--

         if (slowpath(carry)) {
            goto underflow;
        }
    } while (slowpath(!StoreReleaseExclusive(&isa.bits, 
                                             oldisa.bits, newisa.bits)));
    return false;

underflow:
    newisa = oldisa;

    if (slowpath(newisa.deallocating)) {
        ClearExclusive(&isa.bits);
        return overrelease_error();
    }
    newisa.deallocating = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;

    __sync_synchronize();
    if (performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return true;
}
{% endhighlight %}

当引用计数为0，release会产生下溢出，此时转移到`underflow`，没有额外的引用计数存储散列表。此时会标记isa的`deallocating`为1，这样保证了dealloc方法只会被执行一次。最后调用对象的dealloc方法，释放内存。

### 溢出

当引用计数为0，进行release会产生下溢出，但是有额外的引用计数存储在散列表中，下面是从散列表中获取额外引用计数的过程：

{% highlight cpp %}
bool objc_object::rootRelease_underflow(bool performDealloc) {
    return rootRelease(performDealloc, true);
}

size_t objc_object::sidetable_subExtraRC_nolock(size_t delta_rc) {
    assert(isa.nonpointer);
    SideTable& table = SideTables()[this];

    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()  ||  it->second == 0) {
        return 0;
    }
    size_t oldRefcnt = it->second;

    assert((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);
    assert((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);

    size_t newRefcnt = oldRefcnt - (delta_rc << SIDE_TABLE_RC_SHIFT);
    assert(oldRefcnt > newRefcnt);  // shouldn't underflow
    it->second = newRefcnt;
    return delta_rc;
}

bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow) {
    if (isTaggedPointer()) return false;

    isa_t oldisa;
    isa_t newisa;
retry:
    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;

        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--

         if (slowpath(carry)) {
            goto underflow;
        }
    } while (slowpath(!StoreReleaseExclusive(&isa.bits, 
                                             oldisa.bits, newisa.bits)));
    return false;

underflow:
    newisa = oldisa;

   if (slowpath(newisa.has_sidetable_rc)) {
        if (!handleUnderflow) {
            ClearExclusive(&isa.bits);
            return rootRelease_underflow(performDealloc);
        }

        size_t borrowed = sidetable_subExtraRC_nolock(RC_HALF);
        if (borrowed > 0) {
            newisa.extra_rc = borrowed - 1;  // redo the original decrement too
            bool stored = StoreReleaseExclusive(&isa.bits, 
                                                oldisa.bits, newisa.bits);
            if (!stored) {
                isa_t oldisa2 = LoadExclusive(&isa.bits);
                isa_t newisa2 = oldisa2;
                if (newisa2.nonpointer) {
                    uintptr_t overflow;
                    newisa2.bits = 
                        addc(newisa2.bits, RC_ONE * (borrowed-1), 0, &overflow);
                    if (!overflow) {
                        stored = StoreReleaseExclusive(&isa.bits, oldisa2.bits, 
                                                       newisa2.bits);
                    }
                }
            }

            if (!stored) {
                sidetable_addExtraRC_nolock(borrowed);
                goto retry;
            }

            sidetable_unlock();
            return false;
        }
        else {
            // Side table is empty after all. Fall-through to the dealloc path.
        }
   }
}
{% endhighlight %}

和之前retain一样，处理散列表中额外引用计数时依然绕了一个弯子，通过`rootRelease_underflow`方法第二次调用`rootRelease(bool,bool)`方法。

通过`sidetable_subExtraRC_nolock`方法去散列表中验证是否可以拿出额外的128引用计数用来更新isa的`extra_rc`，如果不够则返回0，否则更新散列表中存储的额外引用计数。

判断borrowed是否大于0，如果等于0则直接走到之前到释放内存流程。否则使用borrowed-1的值更新isa的`extra_rc`，因为之前的`extra_rc`为0，还没有减1，所以这里需要进行减1操作。使用newisa的bits更新isa，如果没成功，则会进行两步重试。

这里其实有一个问题，在之前retain出现溢出的时候，我们设置了isa的`has_sidetable_rc`为1，但是在`sidetable_subExtraRC_nolock`更新额外引用计数时，当取完时并没有进行更新`has_sidetable_rc`为0。根据Apple在源码中的注释知道，这里主要为了多线程的竞争，所以一直保持为1。

## 获取引用计数

说完了引用计数的存储，操作引用计数，最后也是最简单的就是获取引用计数了，其实现方法在`objc_object`的`rootRetainCount()` 方法中，其实获取方法就是`1+extra_rc+side_table_rc`，笔者这里就不贴代码了。

## 总结

本文主要根据runtime源码，了解了OC中引用计数的基本实现。