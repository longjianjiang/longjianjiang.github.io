---
layout: post
title:  "【Reative】学习笔记第二弹"
date:   2018-07-16
excerpt:  "本文是笔者学习Reative的第二篇文章，主要介绍Observable的各种操作"
tag:
- Reative
comments: true
---


本文主要介绍【Reative】中Observable的变换操作，笔者认为理解这些操作对后面将Observable应用到实际工程中有很大帮助，因为实际工程中的业务都是通过简单Observable进行不断变换所抽象出来的。

## Transfrom Operator

#### `map`

- map

map 顾名思义， 将Observable中的元素放入map的block中处理，返回一个新的Observable。

```
let signal = SignalProducer<Int, NoError> { (innerObserver, _) in
    innerObserver.send(value: 5)
    innerObserver.send(value: 7)
    innerObserver.sendCompleted()
}

signal.map { $0 * 10}
   .startWithValues { print("map valu \($0)") }
```

转换图如下所示：

```
5 -> 7 -> end

$0 * 10

50 -> 70 -> end
```

> 下面是map的一些便捷方法

- mapWithIndex

此方法可以在遍历的时候按条件选择是否需要map。

> `mapWithIndex` 在新版本中的RxSwift中提示让我们使用 enumerated().map() 替换，效果还是一样的

```
 Observable<Int>.of(1, 2, 3, 4, 5)
            .enumerated()
            .map {index, number in
                number % 2 == 1 ? number * 2 : number
            }
            .subscribe(onNext: { print($0)})
            .disposed(by: bag)
```

转换图如下所示：

```
1 -> 2 -> 3 -> 4 -> 5 -> end

number % 2 == 1 ? number * 2 : number

2 -> 2 -> 6 -> 4 -> 10 -> end
```

- toArray

此方法可以将Observable元素转成用一个数组包装

```
Observable<Int>.of(5, 7)
    .toArray()
    .subscribe(onNext: { print($0 )})
    .disposed(by: bag)
```

转换图如下所示：

```
5 -> 7 -> end

toArray

[5, 7] -> end
```

#### flatMap




## Fliter Operator

## Combine Operator
