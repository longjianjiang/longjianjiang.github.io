---
layout: post
title:  "Swift 笔记"
date:   2019-05-13
excerpt:  "本文是笔者记录的一些从开源代码中看到的一些swift语言特性"
tag:
- Swift
comments: true
---

本文笔者记录下自己从开源代码中看到的一些Swift语言特性，这样更容易去应用到自己写的代码中。

# Trailing Closures

先上一段代码：

{% highlight swift %}
public func dataTask(_: PMKNamespacer, with convertible: URLRequestConvertible) -> Promise<(data: Data, response: URLResponse)> {
    return Promise { dataTask(with: convertible.pmkRequest, completionHandler: adapter($0)).resume() }
}
{% endhighlight %}

笔者刚开始看到上面代码中返回Promise这个还没看明白，初始化怎么直接{},后来想到了这个所谓的TC。

这个特性用的还是比较多的，不过知道了也就习以为常了。

# @autoclosure

先上一段代码：

{% highlight swift %}
func logIfTrue(@autoclosure predicate: () -> Bool) {
    if predicate() {
        print("True")
    }
}

logIfTrue(5 > 7)
{% endhighlight %}

可以看到加了`@autoclosure`后，对于这种closure，我们直接传了一个表达式，编译器会帮我们转成closure的形式。

[ref](https://swifter.tips/autoclosure/)

# enum 

## Associated Values

swift 中的枚举的一个好用的特性就在于可以添加所谓的AV，很多开源库都使用了这个特性。

这个特性允许给枚举的case增加一些需要的值来完成特定的需求，比如一个常见的`Result`枚举：

{% highlight swift %}
enum Result<T> {
	case success(T)
	case failure(Error)
}
{% endhighlight %}

## match

先上一段代码：

{% highlight swift %}
enum Sealant<R> {
    case pending(Handlers<R>)
    case resolved(R)
}

guard case .pending(let _handlers) = self.sealant else {
	return  // already fulfilled!
}
{% endhighlight %}

笔者开始看到guard里加case不知道是干嘛的，后来查了下，其实就是类似`if let`来去除optional这种效果，这里尝试将枚举中的case转为一个特定的case。

不仅仅可以使用上面的`guard case`，还可以使用`if case`和`for case`，具体可以参考[这里](http://alisoftware.github.io/swift/pattern-matching/2016/05/16/pattern-matching-4/)。

# where

在范型中，添加where子句来检查参数类型是否符合条件。

之前说到的枚举中使用`for case`也可以使用where子句来添加限定条件的。

# fatalError

先看一个例子：

{% highlight swift %}
class Box<T> {
    func inspect() -> Sealant<T> { fatalError() }
    func inspect(_: (Sealant<T>) -> Void) { fatalError() }
    func seal(_: T) {}
}
{% endhighlight %}

定义了一个Box类，其中两个`inspect`方法调用了`fatalError`，其实可以猜到是父类给出的方法不希望被调用，需要子类自己去实现。

`fatalError`调用后程序中止，给出设定的提示信息。

# References 

[https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html](https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html)

{% highlight swift %}
{% endhighlight %}
