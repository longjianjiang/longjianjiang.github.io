---
layout: post
title:  "【Runtime源码】类的结构"
date:   2018-12-21
excerpt:  "本文是笔者阅读Runtime源码关于类的结构的笔记"
tag:
- SourceCode
comments: true
---

> 本文笔者根据 Runtime 750 源码分析OC中类的具体结构。

OC中的对象都是 `objc_object` 结构体，OC中的类都是 `objc_class` 结构体，`objc_class` 又继承自 `objc_object`，所以OC中大家都是对象。

