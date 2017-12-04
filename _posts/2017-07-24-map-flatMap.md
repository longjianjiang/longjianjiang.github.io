---
layout: post
title:  "【Tips】map & flatMap"
date:   2017-07-24
excerpt: "在RxSwift中，`map` , `flatMap`两个操作符用的比较算是比较频繁的，开始的时候对`flatMap`理解并不是那么深刻，下面就一个例子来说下自己对于`flatMap`的认识："
tag:
- Swift
comments: true
---

## 前言
在RxSwift中，`map` , `flatMap`两个操作符用的比较算是比较频繁的，开始的时候对`flatMap`理解并不是那么深刻，下面就一个例子来说下自己对于`flatMap`的认识：

```
 Observable.of(1, 2, nil ,4)
        .flatMap { $0 == nil ? Observable.empty() : Observable.just($0!) }
        .subscribe(onNext: {
            print($0)
        })
```

## 开始
其实说到`flatMap`，它相对于`map`的不同就是所谓的**降维**；但是如果用降维来解释如上的栗子，好像并不那么贴切？
> 或许是笔者理解不够透彻，希望读者可以给出自己的解释

## 接着
因为栗子中的`observable`并不是所谓的`inner observables`；栗子中的输出结果是:
 ```
1
2
4
```
如果`flatMap` 换成`map`，输出结果如下：
```
RxSwift.Just<Swift.Int>
RxSwift.Just<Swift.Int>
RxSwift.Empty<Swift.Int>
RxSwift.Just<Swift.Int>
```
结果截然不同！所以下面笔者说下自己的理解：
`map` 和 `flatMap`，两个操作符都是将`observable`中的元素进行变换。`map`操作符变换后的元素类型就是闭包返回的类型，所以本文栗子中使用`map`后，订阅输出的就是`RxSwift.%%`类型；而`flatMap`闭包返回的类型都是`Observable`类型，但是变换后的元素是`Observable`类型中`Element`的类型，所以栗子中使用`flatMap`后输出的依然是 `Int`类型。
栗子中`flatMap`闭包每次返回的`Observable`将其中的`element`发送到一个新的`Observable`，这个新的`Observable`会被订阅者所订阅，这个新的`Observable`就可以说明`flatMap`的降维，也是所谓的`flat`。

## 然后
因为栗子中最初的`Observable`中有一个元素为空，所以`Observable`中`Element`类型应该是`Optional`，但是经过`flatMap`后输出的却不是，说明`flatMap`可以过滤`Observable`中为空的`element`。
之所以能过滤空的`element`，主要还是因为`flatMap`会新建一个`Observable`，因为栗子中闭包，当元素为空的时候返回的是一个空的`Observable`，所以新的`Observable`并不会接收到其中的`element`，之后订阅者所输出的也就不存在空的元素，所以类型自然也就不是`Optional`。

> Swift中的`flatMap`，同样的也可以过滤空的元素

 ```
["ab", "cc", nil, "dd"].flatMap { $0 }
["ab", "cc", nil, "dd"].filter { $0 != nil} .map { $0! }
```
此时数组是`["ab", "cc", "dd"]`,而且不是`Optional`类型，只是使用`flatMap`更佳简洁高效；因为内部使用了` if let`,所以达到了过滤空元素的效果。

# Functor and Monad 更新于2017-12-04

Functor and Monad是函数式编程的重要概念，笔者之前也不是太理解，只是听过名称而已，其实他们的定义和map，flatMap关系很大：

> flatMap其实是一种特殊的map，所谓的降维是因为flat的原因;

下面是笔者对于数组类型实现的map, flatten, flattenMap:

```
extension Array {
    func myMap<T>(_ transform: (Element) -> T) -> [T] {
        var result: [T] = []
        result.reserveCapacity(count)
        for x in self {
            result.append(transform(x))
        }
        return result
    }
    
    func myFlatten<T>(elements: [[T?]]) -> [T] {
        var result: [T] = []
        
        for arr in elements {
            for item in arr {
                if let num = item {
                    result.append(num)
                }
            }
        }

        return result
    }
    
    func myFlattenMap<T>(_ transform: (Element) -> [T?]) -> [T] {
        return myFlatten(elements: myMap(transform))
    }
    
}
```

根据上面的代码，其实functor和monad的定义也就很容易理解了：

> Functor: `map` 函数接受一个闭包作为参数，作用与容器类型（数组，optional，result），通过传入的闭包改变容器类型内部的值，从而得到一个全新的`容器`; 所谓functor就是实现了map功能的类型；

> Monad: monad 是functor中的一种，相比functor不仅实现了map功能，而且实现了flattenMap的功能；


