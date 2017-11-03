---
layout: post
title:  "【Tips】Swift Protocol"
date:   2017-11-3
excerpt:  "本文是笔者学习Swift协议的笔记。"
tag:
- Swift
comments: true
---

### 前言

本文是笔者学习Swift协议的笔记。

### 开始

OC中判断是否遵守某个协议有对应的方法，Swift中也有，但是开始使用的时候遇到了问题：

```
self.conforms(to: LJCacheKitTableProtocol) 
error: Cannot convert value of type 'LJCacheKitTableProtocol.Protocol' to expected argument type 'Protocol'
```

后来发现使用conforms方法来判断是否遵守某个协议时，此时协议必须标记为@objc；

但是Swift中提供了其他的方式来判断是否遵守某个协议：

```
is ： 返回true或false
as? :  返回一个optional类型的遵守该协议的对象，如果为nil则不遵守该协议
as! : 返回一个遵守该协议的对象，如果不遵守则出错
```

默认的 Swift中协议中方法是必须实现的，如果想使用OC中optional类型协议方法，也必须标记协议为@objc。

针对这种情况，如果我们在设计的协议中想要增加一些optional类型的方法，其实可以不用标记协议为@objc，换种思路，可以重新设计协议，讲其拆分为多个，根据需要去选择遵守那个，同时Swift中支持Protocol Composition来组合多个协议。


以下是笔者的笔记：

```
property in protocol {get set};
method in protocol (mutating in enum and struct & init method in protocol);
protocol can use as type like String type;
use protocol implement delegate pattern;
use extension to conform protocol;
protocol can be inherited like class;
use AnyObject to create class-only protocol;
New: use & combine protocol and class type to make a new type( conform several different protocols and inherited from class);
use as(? !) or is to check object is conform some protocol, is just to get true or false , as can downcast;
mark protocol and methods in protocol as @objc can make these methods optional ;
protocol can add extension to provide default implemention , use where to add constraints to protocol extension;
```

#### 最后

未完待续
