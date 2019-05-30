---
layout: post
title: 【Reactive】学习笔记第五弹(disposable)
date: 2019-05-30
excerpt: "nope"
tag:
- Reactive
comments: true
---

```




								+-------------+ 			  +--------------+
								| DisposeBase |				  | <<interface>>|
								+-------------+ 			  |  Disposable  |
															  +--------------+


						    +---------------------+           +--------------+
							| AnonymousDisposable | 		  | <<interface>>|
							+---------------------+ 		  |  Cancelable  |
															  +--------------+

	
		


```

# disposable

当我们用Observabel创建一个信号的时候，总是需要返回一个disposable，用来做一些销毁工作。

实际做这个销毁工作的依然是一个block，对应一个私有类`AnonymousDisposable`。

实际我们创建一个disposable的时候，通常使用`Disposables`提供的接口，这些接口对应根据参数列表的不同实际创建的是不同类型的disposable。此外还有一些公开的disposable可以使用。

# dispose

subscribe方法会返回一个disposable，通常不会对返回的这个disposable进行dispose，而使用`DisposeBag`。`DisposeBag`有一个数组存储了所有的disposable，当bag销毁的时候，会将数组中所有的disposable进行dispose。

> 也可以使用`takeUntil`。


