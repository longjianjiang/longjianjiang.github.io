---
layout: post
title: "isa-swizzling 笔记" 
date:   2019-04-28
excerpt:  "本文是笔者学习isa-swizzling的笔记"
tag:
- iOS
comments: true
---

> 本文笔者以isa-swizzling为引子，介绍ZeroingWeakRef和KVO；

isa-swizzling 根据名字就能猜到是干嘛的，动态的改变object的isa指针，指向另一个class。用它可以干嘛的呢，下面笔者给出两个实际应用：

## ZeroingWeakRef

当我们使用为分类中增加关联对象的时候，`objc_setAssociatedObject`中`policy`枚举中是没有weak选项的。假设我们一个View的一个分类中增加了一个delegate，控制器作为这个View的delegate，此时如果控制器销毁了，View中再去访问这个delegate则会出现crash，因为访问了一个已经销毁的内存地址。所以这个时候一种类似weak的机制来将delegate置空，这样就不会发生crash。

下面笔者介绍mike ash提供的一种通用的解决方案 ZeroingWeakRef，因为那个时候还不是ARC。先看使用效果:

{% highlight objc %}
MAZeroingWeakRef *ref = [[MAZeroingWeakRef alloc] initWithTarget: object];
NSLog(@"Target is %@", [ref target]);
{% endhighlight %}

{% highlight cpp %}
weak_ptr<int> weak_p(shared_p);
auto ref = weak_p.lock();
{% endhighlight %}

使用起来类似C++11中的`weak_ptr`，初始化使用一个对象，下次需要使用这个对象的时候调用另一个方法来获得存储的对象，当该对象被释放的时候，获取方法会返回空，这样即可避免访问到野指针。

那么 ZeroingWeakRef 是如何实现的呢？ 主要利用了isa-swizzling，下面笔者简单介绍其实现：

首先创建一个全局的`gObjectWeakRefsMap`，用来存储所有对象的弱引用关系，key为obj，key是一个set。

当调用`initWithTarget`时，以该对象的类动态创建一个该类的子类，同时将obj的isa指针指向新创建的子类。该子类重写dealloc方法，在dealloc方法中，首先会根据obj和`gObjectWeakRefsMap`找到存储该对象所有弱引用的set，对该set中的所有对象进行置空。最后调用父类的dealloc方法。

不过实际的实现要复杂一些，具体可以参见[源码](https://github.com/mikeash/MAZeroingWeakRef)。

不过现在有了__weak，实现起来就非常简单了，因为__weak帮我们实现了置空的工作，所以我们只需要使用一个weak的property即可。

最后笔者想提一下NSTimer的循环引用，下面代码肯定是会导致对象不能释放的:

{% highlight objc %}
self.timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(timerMethod) userInfo:nil repeats:YES];
{% endhighlight %}

最简单的方法是创建一个代理对象，继承自`NSProxy`,包含一个weak的propety即可，这样timer引用的不是self，而是一个代理对象，这样没有增加self的引用计数，self可以正常销毁。具体实现可以参考[这里](https://github.com/ibireme/YYKit/blob/master/YYKit/Utility/YYWeakProxy.m)。

再看另一种方式是使用block，参考代码如下:

{% highlight objc %}
@implementation NSTimer (Weak)
+ (instancetype)weak_scheduleTimerWithTimeInterval:(NSTimeInterval)timeInterval repeats:(BOOL)repeats block:(void (^)(void))block {
    return [self scheduledTimerWithTimeInterval:timeInterval target:self selector:@selector(timerMethod:) userInfo:[block copy] repeats:repeats];
}

+ (void)timerMethod:(NSTimer *)timer {
    void(^blk)(void) = timer.userInfo;
    if (blk) {
        blk();
    }
}
@end
{% endhighlight %}

通过分类中的weak方法，target指定为了类对象，将timer的方法以block的方式传递存储到userInfo，timer执行的时候去执行分类中的另一个类方法，取之前存的block进行执行。使用起来如下所示:

{% highlight objc %}
 __weak typeof(self) weakSelf = self;
self.timer = [NSTimer weak_scheduleTimerWithTimeInterval:1 repeats:YES block:^{
    [weakSelf timerMethod];
}];
{% endhighlight %}

使用到weak/strong，虽然这里self没有直接持有block，但是通过timer的block去捕获到self依然会导致循环引用，所以需要weak。

这种方式同样可以让self释放，但是却引入了一个新的问题，导致NSTimer类自身循环引用，所以timer的这个block依然会被一直调用，虽然此时weakSelf为nil。

```
不过笔者具体还没想明白怎么解释类自身循环引用。
```

## KVO

KVO的实现其实也是使用到isa-swizzling，对象被某个观察者观察的时候，这个时候同样会动态创建一个该对象类的子类，重写property的set方法，在set方法里面调用`- willChangeValueForKey:`,`- didChangeValueForKey:`。这个时候将该对象的isa指向动态创建的这个类，这样property改变的时候可以通知到观察者。

更多可以参考[这里](http://www.pluto-y.com/isa-swizzling-and-runtime/)和[这里](http://blog.sunnyxx.com/2014/03/09/objc_kvo_secret/)。

## 总结

本文主要介绍了isa-swizzling的相关内容。

## References

[https://mikeash.com/pyblog/friday-qa-2010-07-16-zeroing-weak-references-in-objective-c.html](https://mikeash.com/pyblog/friday-qa-2010-07-16-zeroing-weak-references-in-objective-c.html)
