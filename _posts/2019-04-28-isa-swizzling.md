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

首先创建一个全局的`gObjectWeakRefsMap`，用来存储所有对象的弱引用关系，key为obj，value是一个set。

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

今天看到Effective Objc书上的关于Timer的提到这个问题，当使用这种方式来避免循环引用时，需要在用到timer的类里的dealloc方法中，调用timer的`invalidate`方法，这样定时器就不会继续触发。对于self来说，因为是NSTimer的类实例，本身是一个单例，不会被回收，计时器是否会引用其实没什么影响。

Timer 中新的block的api方法，内部应该也是这样实现的，target设置了Timer类本身。

## KVO

KVO的实现其实也是使用到isa-swizzling，对象被某个观察者观察的时候，这个时候同样会动态创建一个该对象类的子类，重写property的set方法，在set方法里面调用`- willChangeValueForKey:`,`- didChangeValueForKey:`。这个时候将该对象的isa指向动态创建的这个类，这样property改变的时候可以通知到观察者。

那么当property改变的时候是如何通知到观察者的呢？

首先在addObserver方法中，肯定需要将observer进行存起来，一种可能将某个对象的所有observer存储在一个全局的table中，也有可能将某个对象的所有observer存储在动态生成类中，`objc_allocateClassPair`中指定`extraBytes`，这样table就可以存储在类中。

存储了所有的observer后，接下来要做的事情就是通知到observer，一个最简单的方法就是调用observer的`observeValueForKeyPath:ofObject:change:context:`方法，这样observer就会收到通知。所以如果observer的类没有实现该方法，则会走到NSObject的该方法，此时会抛出异常。

最后在observer释放内存之前，需要将其移出，否则对一个释放内存的对象发送消息会出错。

因为KVO接口设计的不够好用，所以有了[MAKVONotificationCenter](https://github.com/mikeash/MAKVONotificationCenter),[KVOController](https://github.com/facebook/KVOController)对原有KVO接口进行封装。

KVOController是以observer为receiver进行封装，通过`observe:keypath:`系列方法以监听者的身份来监听某个对象。和KVO通过`addObserver:forKeypath:`，对象添加监听者正好相反。内部创建了一个私有类专门用来作为监听者，这样当监听者收到KVO通知，通过context拿到本次监听的数据进行转发，block的方式，action的方式，或者`observeValueForKeyPath:ofObject:`的方式。

MAKVONotificationCenter的实现依然是内部创建了一个私有类作为监听者，当收到KVO通知时使用action或block的方式进行转发给外界。

下面笔者提几个之前没有注意的点：

### NSKeyValueObservingOptionPrior

正常情况下，是在被观察属性的值改变后通过didChangeValueForKey:方法触发 KVO 消息。若在注册观察者时添加NSKeyValueObservingOptionPrior选项，则在属性值改变前，即在willChangeValueForKey:方法中也会触发一次 KVO 消息。此时，被观察属性值的改变会触发两次 KVO 消息，可以通过change中的NSKeyValueChangeNotificationIsPriorKey进行区分。
### context

kvo的api中有一个context参数，使用静态变量的地址作为这个参数可以方便高效（相比keypath的字符串比较）来区分不同的监听者。
### 手动通知

有的时候想手动控制发送通知给观察者，需要重载NSObject的`automaticallyNotifiesObserversForKey：`方法，将需要手动控制的key返回NO。

当需要发送通知给观察者的时候，将proepty的改变放在`- willChangeValueForKey:`,`- didChangeValueForKey:`中间即可。

### dependent keys

可以设定某个propety依赖其他若干属性，当这些若干属性改变的时候，property的对象就会发送通知给观察者。

- To-One 

比如fullName依赖，firstName和lastName。这样通过重载`keyPathsForValuesAffectingValueForKey:` 可以指定这种依赖关系。

{% highlight objc %}
+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key {
 
    NSSet *keyPaths = [super keyPathsForValuesAffectingValueForKey:key];
 
    if ([key isEqualToString:@"fullName"]) {
        NSArray *affectingKeys = @[@"lastName", @"firstName"];
        keyPaths = [keyPaths setByAddingObjectsFromArray:affectingKeys];
    }
    return keyPaths;
}
{% endhighlight %}

- To-Many

比如班级的平均身高，依赖班级每个学生的高度属性，此时不能通过`keyPathsForValuesAffectingValueForKey:` 指定`stu.height`。

只能将班级作为每个学生的观察者，来进行统计平均身高，当新的学生加入班级或者学生离开班级，都有做对应的addObserver和removeObserver操作来保证平均身高的准确性。

## 隐藏组件细节

我们知道，isa决定了方法的调用，所以有些时候， 对外的暴露的组件不想让外界通过runtime的方式去进行调用，我们也可以通过isa-swizzling进行替换isa。

[ref](https://www.jianshu.com/p/b5c940f6d80d)

## 总结

本文主要介绍了isa-swizzling和KVO的相关内容。

## References

[https://mikeash.com/pyblog/friday-qa-2010-07-16-zeroing-weak-references-in-objective-c.html](https://mikeash.com/pyblog/friday-qa-2010-07-16-zeroing-weak-references-in-objective-c.html)

[https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html](https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)

[http://www.pluto-y.com/isa-swizzling-and-runtime/](http://www.pluto-y.com/isa-swizzling-and-runtime/)

[KVO](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOImplementation.html#//apple_ref/doc/uid/20002307-BAJEAIEE)
