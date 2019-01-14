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

我们知道OC类中不能存在两个相同名字的方法，参数类型不一样也是不允许的，但是不同类中有相同的名字的方法确是允许的，不过不知道是否想过不同类中的同名方法是不是指向的是同一个SEL呢？

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

之前笔者在类的结构中还剩最后一个cache没有介绍，这个cache就和今天的方法寻找有关系。其实都能猜到，这个cache就是给寻找IMP使用的，在去类中方法列表寻找之前，首先去cache中寻找，而且每次调用完一个方法后将这次调用缓存起来，方便下次寻找。

{% highlight cpp %}
using MethodCacheIMP = IMP;
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

## Dynamic Method Resolution

## Forwarding