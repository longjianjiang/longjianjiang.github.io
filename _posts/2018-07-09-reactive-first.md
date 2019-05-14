---
layout: post
title:  "【Reative】学习笔记第一弹(Intro)"
date:   2018-07-09
excerpt:  "本文是笔者学习Reative的第一篇文章，主要介绍一些基本概念"
tag:
- Reative
comments: true
---


> 说起`Reative`，相信大家一定都不陌生，iOS当中的代表应该是 `ReactiveCocoa`,RAC今天看来已经非常基础了，【Reative】系列文章，笔者打算将自己学习的思路记录下来；


不论当前的`RxSwift`, `ReactiveSwift` 或者 之前的`ReactiveCocoa`, 它们的实现是类似的，思想也是共同的，所以当我们学习这种响应式框架的时候，我们开始最重要的是理解其中的一些基本概念，这对后面的实践是有很大帮助的。 下面笔者以自身学习为例，介绍几个基本概念。


## Side Effects

我们可能在学习`Reative`的时候，多少会看到所谓的Side Effects,`比如当信号被订阅的时候副作用就会发生等等类似的句子`。
举个例子，下面代码中的add函数就产生了副作用，因为它在函数执行中改变了全局变量的值。

{% highlight ruby %}
$global_val = 0

def add(a, b)
  $global_val = 2
  a + b 
end

puts add 5, 7
puts "global_val is #{$global_val}"
{% endhighlight %}

所以所谓的副作用就是函数或者程序在执行过程中改变了自身以外的其他人的状态，这就可以叫产生副作用了。
回到`比如当信号被订阅的时候副作用就会发生等等类似的句子`, 因为订阅执行时会改变外部状态，所以此时说产生副作用也是合情合理的。


## Reactive

|			| One 		| Many									|
| :------	| ------: 	| :------: 								|
| Sync		| T 		| Enumerable[T] / AsyncEnumerable[T]	|
| Async 	| Future[T] | Observable[T]	/ AsyncObservable[T]	|

上表给出了我们编程中四种基本的Effects；


> One & Sync
其实这种情况就是我们最常见的一个方法或者一个函数，执行它的时候，当前执行被阻塞，直到该方法返回。
这也是我们所谓的`Imperative`, 也就是命令式。

> One & Async
这种情况也很常见，一般是一个异步的操作，该操作执行需要时间，会在稍后进行回调，此时当前执行不会被阻塞。
比如JS在语言层面就提供这种异步的支持。

> Many & Sync
这种情况其实和第一种情况很像，只是此时返回的值是一个集合。

> Many & Async
这种情况就类似比如我监听一个`Observable`,在某个时间点会收到值到来的通知。
这种情况也就是我们今天所说的`Reactive`, 所谓的响应式。

```
比较Many下：

Sync的Enumerable 是Pull类型的，也就是说我们想要获取某个值，需要主动的去通过`objectAtIndex`类似的方法去取。
Async的Observable是Push类型的，也就是说只要我监听了，只要有新的值过来我就会收到这个新的值。
```

所以响应式最大的特点是主动性。


## Observable

前面所说的响应式之所以叫响应式，主要就是Observable起的作用，也就是所谓的流。
我们可以把流想象成序列，类似Enumerable，不过比Enumerable更加动态，也就是此时的序列是异步的，而且类型会变化。

类型会变化怎么理解，因为是序列那么肯定是可以获取序列中的值的，所谓的变化指的就是获取序列值的时候会有变化。获取序列中的值，我们可以用最简单的getter来说明。


> normal
var obj: () => String

obj() // 序列正常发送，获取到某个字符串

> exception
var obj: () => String

obj() // 此时序列发送异常，所以也就不会获取到字符串

> termination
var obj: () => String

obj() // 此时序列终止，同样的也就不会获取到字符串

这三种情况大家看了应该很熟悉，因为Observable就是按这三种情况来设计的，Observable作为一个序列，在这个序列中可以发送event，因为有之前所说的变化，所以这个event对应的也就是这三种不同的状态。下面即是RxSwift中所定义的Event类型。

{% highlight swift %}
public enum Event<Element> {
    /// Next element is produced.
    case next(Element)

    /// Sequence terminated with an error.
    case error(Swift.Error)

    /// Sequence completed successfully.
    case completed
}
{% endhighlight %}

## Hot & Cold

前面说到的Observable，其实分为Hot和Cold两大类。

Cold Observable 可以理解为已经准备好待发送的一串序列，一旦有人订阅，那么此时序列中的event就会发送，所以订阅者也会不断的收到回调。
Hot Observable 发送event的时间并不一定在有人订阅时，Hot Observable 发送event可以在任意时间，和是否有人订阅没有关系，所以不同订阅者订阅时间的不同，收到的event序列也是不同的。 而之前的Cold Observable 则不论哪个订阅者收到的event序列都是一样的。


## End

以上就是笔者认为`Reative`的一些基本概念，如不完整，请留言告知。

未完待续。

## 2019-05-12 补充

最近笔者对异步做了进一步的认识，发现其实所谓的Reactive其实和异步的关系挺大的。

Reactive的本质其实就是回调函数（callback）：

1.当我们创建一个信号的时候，我们传递了一个callback，此时这个callback被存起来了；

2.当有人订阅的时候，会传入callback，可能有多个，因为有三种事件；

3.订阅这个方法的内部会使用外部传入的事件callback创建一个订阅者，用这个订阅者作为参数去调用创建信号传入的callback。

4.当创建信号的callback中异步事件完成后，订阅者根据不同事件再去回调对应的事件callback

所以可以看到，响应式其实并没有多么神秘，其实就是框架层面帮我们完成了这个callback的调用，这样使用起来更加优雅更加简洁。