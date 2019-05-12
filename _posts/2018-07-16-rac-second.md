---
layout: post
title:  "【Reative】学习笔记第二弹"
date:   2018-07-16
excerpt:  "本文是笔者学习Reative的第二篇文章，主要介绍Observable的各种操作"
tag:
- Reative
comments: true
---

本文主要介绍【Reative】中Observable的Operator，笔者认为理解这些操作对后面将Observable应用到实际工程中有很大帮助，因为实际工程中的业务都是通过简单Observable进行不断变换所抽象出来的。

# Transfrom Operator

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

最后会生成一个新的信号给外界，来发送信号序列中发送出来的值。

根据取信号序列发送出来值规则的不同，产生了若干不同效果的 "flatMap"，下面介绍RxSwift中提供的三种：

- flatMap

merge效果，按信号序列发送值依次进行转发出来。

- flatMapLatest

每次只转发信号序列中最新的信号发送的值，序列中先前信号发送的值会被忽略。

- flatMapFirst

只转发信号序列中第一个信号发送的值。

---

对于第一种merge效果，ReactiveSwift中还允许自定义，也就是说默认merge是允许信号序列中每个信号发送值一次能转发`UInt.max`个，所以为merge的效果，如果每次只允许信号序列中每个信号发送值一次能转发1个，此时也就是concat的效果，必须等信号序列中第一个信号的值转发完后才能继续转发第二个信号的值，以此类推下去。


# Fliter Operator

## ignore

## skip

## take

## distinct

# Combine Operator

