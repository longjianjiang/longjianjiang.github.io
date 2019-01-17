---
layout: post
title:  "【Runtime源码】类的方法及调用过程"
date:   2019-01-08
excerpt:  "本文是笔者阅读Runtime源码关于类的方法和方法结构和方法调用过程的笔记"
tag:
- Runtime
comments: true
---

> 本文笔者根据 Runtime 750 源码分析OC中类的方法结构和方法调用过程。

本文测试类如下所示:

{% highlight objective_c %}
@interface Person : NSObject {
    NSString *_nickName;
}

- (void)say;
@end

@implementation Person
- (void)say {
    NSLog(@"hello world");
}
@end

/*****************************************************************/

@interface Student: Person
@property (nonatomic, assign) NSInteger grade;

- (void)eat;
@end

@implementation Student

- (void)eat {
    NSLog(@"have dinner");
}
@end

/*****************************************************************/

@interface Animal : NSObject
- (void)say;
@end

@implementation Animal

- (void)say {
    NSLog(@"～～");
}

@end
{% endhighlight %}

# Method

其实在之前的类的加载和类的结构(class_data_bits_t)中已经有提到过关于方法的内容，下面笔者首先给出OC类中方法的结构:

{% highlight cpp %}
typedef struct objc_selector *SEL;
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...); 

using MethodListIMP = IMP;

struct method_t {
    SEL name;
    const char *types;
    MethodListIMP imp;
};
{% endhighlight %}

根据结构体中的成员，我们可以看到OC中一个方法有三部分组成，其中 `types` 可以参考官方文档[Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)，下面笔者来介绍SEL和IMP。

## SEL

SEL我们可以看到定义，是一个 `objc_selector` 结构体指针，不过在源码中并没有给出该结构体的定义，用来表示运行时方法的名字，通常我们可以通过 `@selector`来快速获取到一个方法的SEL，也可以通过 `NSSelectorFromString` 方法获得方法的SEL。

我们知道OC类中不能存在两个相同名字的方法，参数类型不一样也是不允许的，但是一个类中可以存在同名的实例方法和类方法，不同类中有相同的名字的方法确是允许的，不过不知道是否想过不同类中的同名方法是不是指向的是同一个SEL呢？

如果我之前没看过源码，还真不确定，不过现在可以确定的是不同类的同名方法指向的是同一个SEL，SEL的地址编译期就已经确定了。

要证明的话，其实笔者在[类的加载](http://www.longjianjiang.com/runtime-source-code-class-load/)中有提到过，类第一次加载时会将方法的 SEL 注册到一个全局的 `NXMapTable` 中，如果可以找到，就不会继续插入相同的了。

## IMP

IMP其实是一个函数指针，也就是方法的实现。OC类中的方法对应的函数指针和C语言的函数没什么区别，只是默认多了两个参数，第一个参数就是消息的发送方，第二个则是刚刚说到的SEL，接下来就是方法的参数列表了。

{% highlight cpp %}
Person *p = [[Person alloc] init];
void (*say)(id, SEL);
say = (void (*)(id, SEL))[p methodForSelector:@selector(say)];
say(p, @selector(say));
{% endhighlight %}

上面一个例子就是通过IMP像C函数一样调用OC类中的方法。

> 所以OC中的方法调用其实就是根据selector去寻找IMP的过程，因为之前说过类中的 rw 中有方法列表，存储的就是 `method_t` 的列表，通过selector找到对应的IMP就完成了方法的调用。

# Message

笔者之前在[Runtime学习笔记](http://www.longjianjiang.com/runtime/)中说到过Runtime的消息机制，本文就根据源码来分析方法调用具体流程。

## Find

首先笔者给出OC中方法寻找的示意图:

![屏幕快照 2017-07-12 11.22.42.png]({{site.url}}/assets/images/blog/runtime_1.png)

{% highlight cpp %}
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
};
{% endhighlight %}

之前笔者在类的结构中还剩最后一个cache没有介绍，这个cache就和方法寻找有关系。其实都能猜到，这个cache就是给寻找IMP使用的，在去类中方法列表寻找之前，首先去cache中寻找，而且每次调用完一个方法后将这次调用缓存起来，方便下次寻找。

{% highlight cpp %}
using MethodCacheIMP = IMP;
typedef unsigned long		uintptr_t;
typedef uintptr_t cache_key_t;
typedef uint32_t mask_t;

struct bucket_t {
private:
    // 摘录的为 X86 下的分支
    cache_key_t _key;
    MethodCacheIMP _imp;
};

struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
};
{% endhighlight %}

`bucket_t` 的两个成员其实就是SEL和IMP，SEL存储的时候只是将类型强转成了 `cache_key_t`。

`cache_t` 则是一个顺序存储的结构，内部使用`calloc` 来分配一段连续的内存空间，而且内部会自动扩容一倍，扩容后为了内存使用并没有保留之前的数据，buckets 为存储区域的首地址，mask表示存储的容量，occupied则表示当前使用的空间。

{% highlight objective_c %}
Person *p = [[Person alloc] init];
[p say];
{% endhighlight %}

下面笔者以上面代码来展开寻找IMP这一过程，最初的调用栈如下图所示:

![runtime_source_code_method_1]({{site.url}}/assets/images/blog/runtime_source_code_method_1.png)

### objc_msgSend

可以看到栈底的方法就是之前我们说过的 objc_msgSend, 不过后面怎么多了 uncached, 我们其实可以猜到是没有在缓存中找到IMP，下面我们就从源代码中寻找答案。

其实类似 objc_msgSend 有多个不同类型，如下所示:

```
objc_msgSend            ： 返回值为 id
objc_msgSend_fpret      ： 返回值为 floating-point
objc_msgSend_fp2ret
objc_msgSend_stret      ： 返回值为 结构体

objc_msgSendSuper       ： 向父类发送消息，也就是首先从父类中寻找IMP，返回值为 id
objc_msgSendSuper_stret ： 向父类发送消息，也就是首先从父类中寻找IMP，返回值为 结构体
objc_msgSendSuper2
objc_msgSendSuper2_stret
```

> Runtime 中关于这部分使用汇编来实现的，笔者这里根据注释和一点汇编知识来分析下过程。

下面我们以 objc_msgSend 为例，来看代码：

```
// objc_msgSend
	ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame

	NilTest	NORMAL

	GetIsaFast NORMAL		// r10 = self->isa
	CacheLookup NORMAL, CALL	// calls IMP on success

	NilTestReturnZero NORMAL

	GetIsaSupport NORMAL

// cache miss: go search the method lists
LCacheMiss:
	// isa still in r10
	jmp	__objc_msgSend_uncached

    END_ENTRY _objc_msgSend
```

可以看到代码中有 `CacheLookup`, 注释也写了如果找到直接调用IMP。如果没有缓存中没有找到IMP，那么去执行 `__objc_msgSend_uncached`。

> NilTest 这个宏就是判断发送者是否为nil的。

```
// __objc_msgSend_uncached
    STATIC_ENTRY __objc_msgSend_uncached
	UNWIND __objc_msgSend_uncached, FrameWithNoSaves

	// r10 is already the class to search
	MethodTableLookup NORMAL	// r11 = IMP
	jmp	*%r11			// goto *imp

	END_ENTRY __objc_msgSend_uncached
```

可以看到 `__objc_msgSend_uncached` 中又去调用了 `MethodTableLookup` 去找IMP，找到后放到了 r11 中，然后 jmp 去执行IMP。

```
// macro MethodTableLookup
...

call	__class_lookupMethodAndLoadCache3

...
```

`MethodTableLookup` 省略了一些代码，只看关键的，call `__class_lookupMethodAndLoadCache3` 方法。

{% highlight cpp %}
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls) {
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
{% endhighlight %}

`_class_lookupMethodAndLoadCache3` 方法在 objc-runtime-new.mm 中，可以看到只是简答的调用了 `lookUpImpOrForward`, 可以看到 cache 传了NO，因为之前在汇编中就已经尝试从缓存中寻找过了，所以这里就没有必要再次寻找了。

### lookUpImpOrForward

`lookUpImpOrForward` 就是一个标准的寻找IMP的过程，下面让我们来看这个方法的实现:

{% highlight cpp %}
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver) {
    IMP imp = nil;
    bool triedResolver = NO;

    runtimeLock.assertUnlocked();

    // Optimistic cache lookup
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }
    ...
}
{% endhighlight %}

这一步则首先尝试从缓存中寻找IMP，`cache_getImp` 方法是用汇编实现的，同样用到了之前说到的 `CacheLookup`宏。汇编实现笔者这里不展开，具体可以去参看源码，注释写的很详细。

{% highlight cpp %}
static bool isKnownClass(Class cls) {
    return (sharedRegionContains(cls) ||
            NXHashMember(allocatedClasses, cls) ||
            dataSegmentsContain(cls));
}

IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver) {
    ...
    runtimeLock.lock();
    checkIsKnownClass(cls);

    if (!cls->isRealized()) {
        realizeClass(cls);
    }

    if (initialize  &&  !cls->isInitialized()) {
        runtimeLock.unlock();
        _class_initialize (_class_getNonMetaClass(cls, inst));
        runtimeLock.lock();
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }
    ...
}
{% endhighlight %}

这一步Runtime会检查这个类是否能被找到，下面尝试对类进行初始化：

- 第一次尝试第一次初始化类，realizeClass 方法。
- 第二次尝试调用类的 `+initialize` 方法。

关于两次初始化，可以参考[类的加载过程](http://www.longjianjiang.com/runtime-source-code-class-load/)。

{% highlight cpp %}
static method_t *findMethodInSortedMethodList(SEL key, const method_list_t *list) {
    const method_t * const first = &list->first;
    const method_t *base = first;
    const method_t *probe;
    uintptr_t keyValue = (uintptr_t)key;
    uint32_t count;
    
    for (count = list->count; count != 0; count >>= 1) {
        probe = base + (count >> 1);
        
        uintptr_t probeValue = (uintptr_t)probe->name;
        
        if (keyValue == probeValue) {
            // `probe` is a match.
            // Rewind looking for the *first* occurrence of this value.
            // This is required for correct category overrides.
            while (probe > first && keyValue == (uintptr_t)probe[-1].name) {
                probe--;
            }
            return (method_t *)probe;
        }
        
        if (keyValue > probeValue) {
            base = probe + 1;
            count--;
        }
    }
    
    return nil;
}

static method_t *search_method_list(const method_list_t *mlist, SEL sel) {
    int methodListIsFixedUp = mlist->isFixedUp();
    int methodListHasExpectedSize = mlist->entsize() == sizeof(method_t);
    
    if (__builtin_expect(methodListIsFixedUp && methodListHasExpectedSize, 1)) {
        return findMethodInSortedMethodList(sel, mlist);
    } else {
        for (auto& meth : *mlist) {
            if (meth.name == sel) return &meth;
        }
    }

    return nil;
}

static method_t *getMethodNoSuper_nolock(Class cls, SEL sel) {
    runtimeLock.assertLocked();

    assert(cls->isRealized());

    for (auto mlists = cls->data()->methods.beginLists(), 
              end = cls->data()->methods.endLists(); 
         mlists != end;
         ++mlists)
    {
        method_t *m = search_method_list(*mlists, sel);
        if (m) return m;
    }

    return nil;
}

IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver) {
    ...
 retry:    
    runtimeLock.assertLocked();

    imp = cache_getImp(cls, sel);
    if (imp) goto done;

    {
        Method meth = getMethodNoSuper_nolock(cls, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, cls);
            imp = meth->imp;
            goto done;
        }
    }
    ...
}
{% endhighlight %}

这一步到了查找阶段，之前已经加锁，注释写了为了让查找和添加缓存成为原子操作，因为可能运行时可能类的分类中会增加新方法。我们看到在查找之前还是尝试了从缓存中取。

遍历类的 `rw` 方法列表(method_array_t)，查找每一个列表(method_list_t), 实际查找的方法实现是在 `search_method_list` 中，首先判断方法列表 `method_list_t` 有没有 fixedUp，这一过程在[类的加载](http://www.longjianjiang.com/runtime-source-code-class-load/)中有叙述, 此时方法列表的元素根据sel的指针地址排序过，所以查找分成了两种情况:

- 方法列表有序，二分查找，实现在 `findMethodInSortedMethodList`
- 方法列表无序，遍历

得到查找方法返回的 meth，判断非空，下面一步就是将该方法缓存下来，这一步的调用栈如下:

![runtime_source_code_method_2]({{site.url}}/assets/images/blog/runtime_source_code_method_2.png)

可以看到最终的实现是方法 `cache_fill_nolock`:

{% highlight cpp %}
static void cache_fill_nolock(Class cls, SEL sel, IMP imp, id receiver) {
    cacheUpdateLock.assertLocked();

    if (!cls->isInitialized()) return;

    if (cache_getImp(cls, sel)) return;

    cache_t *cache = getCache(cls);
    cache_key_t key = getKey(sel);

    mask_t newOccupied = cache->occupied() + 1;
    mask_t capacity = cache->capacity();
    if (cache->isConstantEmptyCache()) {
        cache->reallocate(capacity, capacity ?: INIT_CACHE_SIZE);
    }
    else if (newOccupied <= capacity / 4 * 3) {
        // Cache is less than 3/4 full. Use it as-is.
    }
    else {
        cache->expand();
    }

    bucket_t *bucket = cache->find(key, receiver);
    if (bucket->key() == 0) cache->incrementOccupied();
    bucket->set(key, imp);
}
{% endhighlight %}

首先判断类有没有执行过 `+initialize` 方法，因为多线程的原因，同时判断缓存中有没有存在需要存储的SEL。

存储之前，根据缓存的容量和所用空间，分为三种情况:

- 缓存内容size为0，第一次使用缓存，初始化缓存，默认容量为4
- 缓存内容size小于容量的3/4,正常插入
- 缓存内容size大于容量的3/4，将缓存大小扩大1倍，不会保留原有的数据

找到cache列表中未使用的地址，将cache中 `_occupied` 加1，通过 `bucket->set(key, imp)` 将SEL&IMP存储到cache中。

> 笔者这里省略了cache内部的一些方法实现。

{% highlight cpp %}
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver) {
    ...
 retry:    
     {
        unsigned attempts = unreasonableClassCount();
        for (Class curClass = cls->superclass;
             curClass != nil;
             curClass = curClass->superclass)
        {
            // Halt if there is a cycle in the superclass chain.
            if (--attempts == 0) {
                _objc_fatal("Memory corruption in class list.");
            }
            
            // Superclass cache.
            imp = cache_getImp(curClass, sel);
            if (imp) {
                if (imp != (IMP)_objc_msgForward_impcache) {
                    // Found the method in a superclass. Cache it in this class.
                    log_and_fill_cache(cls, imp, sel, inst, curClass);
                    goto done;
                }
                else {
                    // Found a forward:: entry in a superclass.
                    // Stop searching, but don't cache yet; call method 
                    // resolver for this class first.
                    break;
                }
            }
            
            // Superclass method list.
            Method meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
                log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
                imp = meth->imp;
                goto done;
            }
        }
    }
    ...
}
{% endhighlight %}

上一步在类中如果没有找到的话，此时就需要去父类中寻找。依然和之前在本类中一样，两步走：

- 首先从父类缓存中找
- 然后从父类方法列表中找

每次找到依然将 SEL&IMP 放入缓存。

> 当缓存返回的IMP是 `_objc_msgForward_impcache`, 则终止在父类中的寻找。

## Dynamic Method Resolution

当类及其父类中都没有找到IMP，此时会进入所谓的方法动态决议阶段。

{% highlight cpp %}
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver) {
    bool triedResolver = NO;
    ...
 retry:    
    if (resolver  &&  !triedResolver) {
        runtimeLock.unlock();
        _class_resolveMethod(cls, sel, inst);
        runtimeLock.lock();
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES;
        goto retry;
    }
    ...
}
{% endhighlight %}

当resolver为true时，方法动态决议只会执行一次，具体实现在 `_class_resolveMethod`中。

{% highlight cpp %}
void _class_resolveMethod(Class cls, SEL sel, id inst) {
    if (! cls->isMetaClass()) {
        // try [cls resolveInstanceMethod:sel]
        _class_resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        _class_resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            _class_resolveInstanceMethod(cls, sel, inst);
        }
    }
}
{% endhighlight %}

可以看到首先判断是是否是metaClass，之前说过实例方法存储在类中，而类方法则存储在metaClass中，所以如果sel是实例方法则执行 `_class_resolveInstanceMethod`, 类方法则执行 `_class_resolveClassMethod`。

{% highlight cpp %}
IMP lookUpImpOrNil(Class cls, SEL sel, id inst, 
                   bool initialize, bool cache, bool resolver) {
    IMP imp = lookUpImpOrForward(cls, sel, inst, initialize, cache, resolver);
    if (imp == _objc_msgForward_impcache) return nil;
    else return imp;
}

static void _class_resolveInstanceMethod(Class cls, SEL sel, id inst) {
    if (! lookUpImpOrNil(cls->ISA(), SEL_resolveInstanceMethod, cls, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);

    IMP imp = lookUpImpOrNil(cls, sel, inst, 
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);
}
static void _class_resolveClassMethod(Class cls, SEL sel, id inst)
{
    assert(cls->isMetaClass());

    if (! lookUpImpOrNil(cls, SEL_resolveClassMethod, inst, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(_class_getNonMetaClass(cls, inst), 
                        SEL_resolveClassMethod, sel);

    IMP imp = lookUpImpOrNil(cls, sel, inst, 
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);
}
{% endhighlight %}

`_class_resolveClassMethod`首先判断cls为metaClass，然后 `_class_resolveClassMethod` 和 `_class_resolveInstanceMethod` 首先查找对应的类是否实现了 `+resolveClassMethod` 和 `+resolveInstanceMethod` IMP，不过因为NSObject实现了这两个方法，所以都会找到的，NSObject默认实现仅仅返回了NO。

对应的查找方法 `lookUpImpOrNil` 内部还是调用了之前说过的 `lookUpImpOrForward` 返回一个IMP, 只是当IMP是 `_objc_msgForward_impcache` 返回的是nil。

接着 `_class_resolveClassMethod` 和 `_class_resolveInstanceMethod` 会给cls发送一个msg，也就是调用 `+resolveClassMethod` 和 `+resolveInstanceMethod` 方法，也就是动态的给sel增加一个IMP，结束后再次调用了 `lookUpImpOrNil`，将本次方法决议缓存，不论给sel增加IMP是否成功，下次再调用这个sel就不会再走到方法决议了，因为可以从缓存中找到一个IMP，有可能是 `_objc_msgForward_impcache`。

### 疑问

{% highlight objective_c %}
@interface Person : NSObject {
    NSString *_nickName;
}
+ (void)run;
@end

@implementation Person
void dynamicMethodIMP(id obj, SEL _cmd) {
    NSLog(@"Person class method run");
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(run)) {
        class_addMethod([self class], sel, (IMP)dynamicMethodIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        [Person run];
    }
    return 0;
}
{% endhighlight %}

当是类方法的时候，首先调用了 `_class_resolveClassMethod`，然后尝试寻找sel的IMP，看是否通过`resolveClassMethod`为sel添加了IMP，如果没有添加则继续尝试调用 `_class_resolveInstanceMethod`。

不过笔者做了实验，在`_class_resolveInstanceMethod` 中对cls发送 `+resolveInstanceMethod` 消息时，类中实现了 `+resolveInstanceMethod` 方法，但是并没有走到，而是去了NSObject的 `+resolveInstanceMethod`。

所以其实当是类方法时，类没有给对应的sel增加IMP时，尝试调用 `_class_resolveInstanceMethod` 并没有效果。

## Forwarding

{% highlight cpp %}
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver) {
    ...
 retry:    
    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);
 done:
    runtimeLock.unlock();
    return imp;
    ...
}
{% endhighlight %}

最后一个步骤就是所谓的消息转发，默认会返回一个转发的IMP，同时加入 sel&IMP加入缓存，此时 `lookUpImpOrForward` 整个过程结束，拿到IMP后具体执行又到了汇编代码。

