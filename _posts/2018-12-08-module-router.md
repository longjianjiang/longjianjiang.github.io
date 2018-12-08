---
layout: post
title:  "Router"
date:   2018-12-08
excerpt:  "本文是笔者探索组件化的笔记"
tag:
- Architecture
comments: true
---

> 组件化实施过程中肯定是需要一个Router，本文笔者分析自己对于Router的理解。

笔者之前开发过程中进行界面跳转都是直接通过 push 或者 present 的方式进行，这样各个模块的控制器就会相互依赖，实施组件化就比较困难，所以我们需要一个 Router 来帮我们统一的做跳转和调用。

这个Router我们希望可以在任意地方以任意方式跳转到任意控制器。也就是要求 Router 可以通过某种方式找到对应的控制器。

业界的方式有以下三种:

- 通过 字符串表示一个控制器，比如 URL;
- 通过 Protocol 的方式获取一个符合该 Protocol 的控制器;
- 通过 runtime 使用 target action 的方式直接获得模块提供的控制器;

