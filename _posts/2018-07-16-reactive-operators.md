---
layout: post
title:  "【Reactive】学习笔记第二弹(Operators)"
date:   2018-07-16
excerpt:  "本文是笔者学习Reative的第二篇文章，主要介绍Observable的各种操作"
tag:
- Reactive
comments: true
---

本文主要介绍【Reative】中Observable的Operator，笔者认为理解这些操作对后面将Observable应用到实际工程中有很大帮助，因为实际工程中的业务都是通过简单Observable进行不断变换所抽象出来的。

# Transfrom Operators

## map

- map

map 顾名思义， 将Observable中的元素放入map的block中处理，返回一个新的Observable。

{% highlight swift %}
let signal = SignalProducer<Int, NoError> { (innerObserver, _) in
    innerObserver.send(value: 5)
    innerObserver.send(value: 7)
    innerObserver.sendCompleted()
}

signal.map { $0 * 10}
   .startWithValues { print("map valu \($0)") }
{% endhighlight %}

转换图如下所示：

```
5 -> 7 -> end

$0 * 10

50 -> 70 -> end
```

> 下面是map的一些便捷方法

- mapWithIndex

此方法可以在遍历的时候按条件选择是否需要map。

`mapWithIndex` 在新版本中的RxSwift中提示让我们使用 enumerated().map() 替换，效果还是一样的

{% highlight swift %}
 Observable<Int>.of(1, 2, 3, 4, 5)
            .enumerated()
            .map {index, number in
                number % 2 == 1 ? number * 2 : number
            }
            .subscribe(onNext: { print($0)})
            .disposed(by: bag)
{% endhighlight %}

转换图如下所示：

```
1 -> 2 -> 3 -> 4 -> 5 -> end

number % 2 == 1 ? number * 2 : number

2 -> 2 -> 6 -> 4 -> 10 -> end
```

- toArray

此方法可以将Observable元素转成用一个数组包装

{% highlight swift %}
Observable<Int>.of(5, 7)
    .toArray()
    .subscribe(onNext: { print($0 )})
    .disposed(by: bag)
{% endhighlight %}

转换图如下所示：

```
5 -> 7 -> end

toArray

[5, 7] -> end
```

## flatMap

flatMap 是一种特殊的Map，特别之处在于flatMap的block参数返回的是一个信号，而map的block参数的是信号发送值的类型。

不论原信号发送值的类型，经过上一步操作后，原信号会发送一个信号序列。

最后会生成一个新的信号给外界，来转发信号序列中发送出来的值。

根据取信号序列发送出来值规则的不同，产生了若干不同效果的 "flatMap"，下面介绍RxSwift中提供的三种：

- flatMap

merge效果，按信号序列发送值依次进行转发出来。

- flatMapLatest

每次只转发信号序列中最新的信号发送的值，序列中先前信号发送的值会被忽略。

- flatMapFirst

只转发信号序列中第一个信号发送的值。

---

对于第一种merge效果，ReactiveSwift中还允许自定义，也就是说默认merge是允许信号序列中每个信号发送值一次能转发`UInt.max`个，所以为merge的效果，如果每次只允许信号序列中每个信号发送值一次能转发1个，此时也就是concat的效果，必须等信号序列中第一个信号的值转发完后才能继续转发第二个信号的值，以此类推下去。


# Fliter Operators

## ignoreElements

ignoreElements顾名思义，忽略信号中发送的next event，但会转发错误和完成。

## skip

skip 会忽略信号中发送的一些next event。

- skip 

会跳过信号发送的前n个next event。

- skipWhile

当给定条件为false 后开始正常发送next event，后面给定条件为true也正常发送。

- skipUntil

忽略原信号发送的next event，直到第二个信号发送next event后，原信号发送的next event正常被发送。

## take

take 会取信号中发送的若干个next event，和skip效果相反。

- take

取信号发送的前n个next event。

- takeWhile

当给定条件为false后，后面的next event将不会被发送，后面给定条件为true也不会发送。

- takeUntil

原信号next event正常被发送，直到第二个信号发送next event后，原信号发送的next event被忽略。

## elementAt

elementAt 只发送信号发送的第n个next event。

> n从0开始计数。

## filter

filter 给定一个条件，只发送信号中符合条件的next event。

## distinctUntilChanged

过滤信号中发送的重复的next evnet。

默认是根据`Equatable`协议来比较是否相等，不过RxSwift中提供了自定义比较方法的接口。

# Combine Operators

## startWith

在信号发送之前，先发送一个给定的next event。

## concat

连接两个信号的next event。

## merge

合并两个信号的next event。

## CombineLatest

组合两个信号发送的next event，如下图所示：

![reactive_second_1]({{site.url}}/assets/images/blog/reactive_second_1.png)

## zip

![reactive_second_2]({{site.url}}/assets/images/blog/reactive_second_2.png)

可以看到zip和combineLatest的区别在于是否使用先前的event。

## withLatestFrom

![reactive_second_3]({{site.url}}/assets/images/blog/reactive_second_3.png)

当一个信号触发后，发送另一个信号当前最新的event。

## sample

![reactive_second_4]({{site.url}}/assets/images/blog/reactive_second_4.png)

可以看到sample和withLatestFrom的区别在于，不会重复发送相同的event。

而且withLatestFrom中，触发器作为调用者；sample中，触发器作为参数。

## amb （ambiguous）

![reactive_second_5]({{site.url}}/assets/images/blog/reactive_second_5.png)

只会转发信号序列中第一个发送event的信号。

## switchLatest

![reactive_second_6]({{site.url}}/assets/images/blog/reactive_second_6.png)

效果类似flatMapLatest。只发送信息序列中最新信号发送的event。

## reduce

![reactive_second_7]({{site.url}}/assets/images/blog/reactive_second_7.png)

reduce 将信号中发送的所有event依次执行一个方法，最后只发送一个结果。

## scan

![reactive_second_8]({{site.url}}/assets/images/blog/reactive_second_8.png)

类似reduce，将reduce的计算结果全部发送，而不是仅仅发送最后的结果。

# Connectable Operators

当订阅的Observable执行有一定代价的时候，我们并不想每一次订阅都去执行一次操作，这个时候就需要用到`multicast`，也就是所谓的多播。

举个例子，网络请求使用Observable进行封装，订阅这个Observable，我们想要的效果其实是每次不论多少个订阅者，只发送一次请求，请求完成后将结果进行通知所有的订阅者，这个时候就需要用到connect operator。

这个多播的实现其实用到了Subject，`publish`，`replay`其实就是指定了`PublishSubject`, `ReplaySubject`。

`refCount` 当Observable第一次被订阅的时候，会自动`connect`原先的Observable。

`share` 其实就是前面几个操作符的组合。

{% highlight swift %}
public func share(replay: Int = 0, scope: SubjectLifetimeScope = .whileConnected)
    -> Observable<Element> {
    switch scope {
    case .forever:
        switch replay {
        case 0: return self.multicast(PublishSubject()).refCount()
        default: return self.multicast(ReplaySubject.create(bufferSize: replay)).refCount()
        }
    case .whileConnected:
        switch replay {
        case 0: return ShareWhileConnected(source: self.asObservable())
        case 1: return ShareReplay1WhileConnected(source: self.asObservable())
        default: return self.multicast(makeSubject: { ReplaySubject.create(bufferSize: replay) }).refCount()
        }
    }
}
{% endhighlight %}

# References

[http://reactivex.io/documentation/operators.html](http://reactivex.io/documentation/operators.html)

[https://store.raywenderlich.com/products/rxswift](https://store.raywenderlich.com/products/rxswift)
