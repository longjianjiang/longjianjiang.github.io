---
layout: post
title:  "【Tips】Swift Protocol"
date:   2017-11-3
excerpt:  "本文是笔者学习Swift协议的笔记。"
tag:
- Swift
comments: true
---

## 前言

本文是笔者学习Swift协议的笔记。

## 开始

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

## POP

所谓面向协议编程。定义一个协议其实就是定义一个接口，这个接口其实就是一种能力，你实现这个接口，就具有这种能力，不关心时何种类型。

### 代码复用

有一块逻辑最初写在某个对象中，但是随着业务发展，发现这个逻辑其他地方也需要，那这个时候可以定一个协议，把这块逻辑给放进协议的默认实现，这样需要使用的对象直接遵守协议就可以使用了。

### 多肽

有些时候同一个逻辑有不一样的实现，这个时候可以为这个逻辑抽象一个接口，不同的实现去遵守协议给出自己的实现，这样使用方只需要依赖这个协议就好，不关心具体的实现。

## 协议的实现

协议用了一个Existential Container的数据结构来存储。

前三个word：Value buffer。用来存储Inline的值，如果word数大于3，则采用指针的方式，在堆上分配对应需要大小的内存
第四个word：Value Witness Table(VWT)。每个类型都对应这样一个表，用来存储值的创建，释放，拷贝等操作函数。(管理 Existential Container 生命周期)
第五个word：Protocol Witness Table(PWT)，用来存储协议的函数。
