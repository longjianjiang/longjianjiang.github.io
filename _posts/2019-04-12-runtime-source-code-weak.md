---
layout: post
title:  "【Runtime源码】weak"
date:   2019-04-12
excerpt:  "本文是笔者阅读Runtime源码关于弱引用实现的笔记"
tag:
- Runtime
comments: true
---

> 本文笔者根据 Runtime 750 源码分析弱引用的实现。

## 开始

这里笔者先提出几个疑问：

1.这里的弱引用不持有是什么意思？

2.所谓强引用持有又是什么意思？

~~3.一个类里面的成员不都应该是属于该类吗？也就是持有所有成员。~~

其实不知道大家有没有发现，之前手动引用计数时期，并没有weak的概念，为什么呢？因为那个时候需要我们自己去写引用计数的方法，所有的一切都交给程序员去控制。而自动引用计数到来之时，之前的那些控制引用计数的方法，由编译器在合适的地方进行插入。那编译器是如何知道在哪里进行插入呢？所以此时有了所谓强引用的概念，默认对象都是强引用。

{% highlight objc %}
id obj = [NSObject new];
id sobj = obj;
id __weak wobj = obj;
{% endhighlight %}

当将一个对象赋值给另一个对象的时候，编译器发现`sobj`是强引用，此时会调用`objc_retain`将`obj`的引用计数加1。编译器发现`wobj`是一个弱引用，此时并不会调用`objc_retain`将`obj`的引用计数加1。

所以所谓强引用和弱引用的区别就在于是否影响对象的引用计数，他们都是指向同一块堆内存，因为使用引用计数的内存管理方式，防止循环引用而出现的一个解决方案而已。

所以所谓的持有其实可以理解为是否引用对象的引用计数。类中成员是否用`__strong`,`__weak`修饰还是一样的道理，当另一个对象`o`用来赋值给类中成员时，如果是强引用则会影响`o`的引用计数，弱引用则不会。这也就是View的delegate属性都设置为`__weak`的原因，如果设置为`__strong`，控制器持有View，而View的delegate会用控制器来赋值，所以View就影响了控制器的引用计数，从而导致了控制器不能释放。

下面来看runtime中如何来记录弱引用关系的，以及如何实现当对象销毁时，对应的弱引用被置为`nil`的。

## 原理

我们都知道使用`__weak`的一个效果就是，右值对象释放时，左值指向该对象的指针会被置为`nil`。其实想一下实现这个机制并不困难，首先每次对弱引用赋值的时候，将右值对象作为key，而对应的value是一个数组，存储的是左值对象地址，因为可能多个弱引用指向同一个右值对象。当对象销毁时，根据对象找到对应的数组，将数组中的指针指向的值置为`nil`即可。

## 数据结构

再看具体实现之前，先看具体使用到的一些数据结构。

{% highlight cpp %}
struct SideTable {
    spinlock_t slock;
    RefcountMap refcnts;
    weak_table_t weak_table;
};
{% endhighlight %}

首先是散列表，存储着对象的弱引用和[额外引用计数](http://www.longjianjiang.com/runtime-source-code-reference-counting/)。

{% highlight cpp %}
#if __LP64__
#define PTR_MINUS_2 62
#else
#define PTR_MINUS_2 30
#endif

#define WEAK_INLINE_COUNT 4
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
};
{% endhighlight %}

`weak_table_t` 存储了所有对象的弱引用关系，而一个对象的弱引用关系则由`weak_entry_t`来存储。`weak_entry_t`中存储左值对象地址的是联合中结构体成员`inline_referrers` 或者 `referrers`。默认先存储在`inline_referrers`的数组中，个数为4，当存满了后，则会存储在`referrers`，并会将原先的4个复制。

使用`referrers`存储时并不是按顺序去将左值对象地址存储到数组中，而是使用hash计算出一个随机索引进行乱序存储的。同时每次存储时判断如果大于容量的`3/4`，则会进行2倍扩容。

上述更新entry的方法在 `objc_weak.mm` 的 `append_referrer` 方法中。

## weak_register_no_lock

当给一个弱引用进行赋值的时候，调用栈如下所示:

![runtime_source_code_weak_1]({{site.url}}/assets/images/blog/runtime_source_code_weak_1.png)

实现存储的就是`storeWeak`方法，这个方法是一个模版方法，提供了三个模版参数:

{% highlight cpp %}
// Template parameters.
enum HaveOld { DontHaveOld = false, DoHaveOld = true };
enum HaveNew { DontHaveNew = false, DoHaveNew = true };
enum CrashIfDeallocating {
    DontCrashIfDeallocating = false, DoCrashIfDeallocating = true
};

template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating>
static id storeWeak(id *location, objc_object *newObj) {
    return (id)newObj;
}
{% endhighlight %}

hasOld表示该弱引用之前有没有旧值，如果有则需要进行清除旧值；

hasNew表示弱引用是否有新值，正常情况下为true，对应的newObj参数有值；如果为false，则newObj为nil;

crashIfDeallocating表示，当注册的时候如果newObj正在销毁，则产生crash；

下面来看添加一个对象的弱引用关系到weak_table中，实现在`weak_register_no_lock`方法中，步骤如下:

{% highlight cpp %}
id weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating) {
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // now remember it and where it is being stored
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        weak_entry_t new_entry(referent, referrer);
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }

    return referent_id;
}
{% endhighlight %}


1.判断newObj非空同时不是taggedPointer；
2.判断newObj没有在释放内存；
3.如果当前右值对象第一次赋值给一个弱引用，此时会初始化weak_table，默认weak_table中`weak_entries`大小为64，根据弱引用赋值表达式的两个参数初始化一个new_entry,将其插入到weak_table。具体插入方式和之前`weak_entry_t`使用地址hash进行获取索引进行插入一样。
4.如果当前右值对象存在一个弱引用，那么通过`weak_entry_for_referent`就可以从weak_table中查到这个entry，直接调用之前所说的更新entry的`append_referrer`方法即可。

目前为止，直接的所说的数据结构关系如下图所示:


## weak_unregister_no_lock

当一个弱引用指向一个新的对象时，需要从weak_table中移除一个对象的弱引用关系。实现在`weak_unregister_no_lock`方法中，步骤如下:

{% highlight cpp %}
void weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
                        id *referrer_id) {
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    weak_entry_t *entry;

    if (!referent) return;

    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        remove_referrer(entry, referrer);
        bool empty = true;
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        }
        else {
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }

        if (empty) {
            weak_entry_remove(weak_table, entry);
        }
    }
}
{% endhighlight %}

1.首先判断右值对象是否为空；
2.通过`weak_entry_for_referent`就可以从weak_table中查到这个entry，从存储旧对象左值地址的数组中移除当前赋值表达式左值的地址；
3.判断旧对象的弱引用关系entry中存储左值地址的数组是否为空，如果为空，则调用`weak_entry_remove`从weak_table中移除旧对象的弱引用关系。


## References

[https://triplecc.github.io/2019/03/20/objective-c-weak-implement/](https://triplecc.github.io/2019/03/20/objective-c-weak-implement/)






