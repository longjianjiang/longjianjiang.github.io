---
layout: post
title:  "【Tips】Swift 中 struct 和 class 选谁？"
date:   2019-01-21
excerpt:  "本文介绍了Swift中新建对象选择struct还是class。"
tag:
- Runtime
comments: true
---

> Swift 中有 class 和 struct，而且 struct 相比传统的C语言中的struct多出了很多功能，使swift中的struct更加像class，这就出现一个问题，怎么选择，当我们创建自定义对象的时候，选择哪一个？

首先给出一个[答案](https://www.mikeash.com/pyblog/friday-qa-2015-07-17-when-to-use-swift-structs-and-classes.html): 当需要值语义就选struct；当需要引用语义，就选class。


