---
layout: post
title:  "设计模式学习笔记"
date:   2018-11-24
excerpt:  "本文是笔者学习设计模式的笔记"
tag:
- Architecture
comments: true
---

## Delegation Pattern

代理模式在iOS中很常见了，`UITableView`的`delegate` 和 `dataSource` 都是属于代理模式。       

其实根据tableView的`delegate` 和 `dataSource`，我们就能知道在什么情况下需要使用代理模式：

```
dataSource: 当需要让外界提供一些数据的时候，可以定义一个protocol，让第三方去实现这个protocol，从而达到获取数据的目的。
delegate: 当需要让外界知道一些动作或者数据的时候，可以定义一个protocol，在对应的地方调用这个protocol，当第三方遵守时，对应的会收到回调。
```

## Strategy Pattern

策略模式，和代理模式有点像。代理模式只有一个第三方对象遵守协议，而策略模式下会有很多个第三方对象遵守同一个协议，此时类可以根据不同的情况选择合适的对象去做一件事情。有时为了避免多个if的判断，也可以使用策略模式。

## Singleton Pattern

单例模式在iOS中也很常见了，苹果很多地方都用到了单例模式，比如`sharedObject`，`defaultObject`等。     

单例模式其实很好理解，有时候为了全局的控制某个类只允许有一个实例，此时我们就可以用单例模式去进行限制。

> 有时候某个类默认提供一个全局的实例，但是同时允许我们创建自己的实例，比如`FileManager`类，一般我们在后台线程会自己创建一个`FileManager`类的实例。

## Memento Pattern

备忘录模式，笔者之前没有接触过这种模式。主要目的是存储类的某个状态同时提供一个方法使得该类未来可以恢复到该状态。    
  
不过这种目的，其实类似于为了不使某个类变得很大很复杂，将一部分功能转移出来类似，这里的功能就是提供一种状态恢复的功能。        

## Observer Pattern

监听者模式，iOS中的KVO就是一种监听者模式，或者使用Rx的第三方框架也可以实现监听者模式。    

## Builder Pattern

