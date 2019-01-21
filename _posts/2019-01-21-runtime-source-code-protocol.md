---
layout: post
title:  "【Runtime源码】类的协议"
date:   2019-01-21
excerpt:  "本文是笔者阅读Runtime源码关于类协议的笔记"
tag:
- Runtime
comments: true
---

> 本文笔者根据 Runtime 750 源码分析OC中类协议的数据结构。

前面笔者介绍过了类的加载过程，不过没有详细的介绍 protocol，本文笔者就着重分析类的协议相关内容，测试类如下所示: