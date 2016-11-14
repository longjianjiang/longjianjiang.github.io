---
layout: post
title:  "【Tips】UIScrollView用autolayout设置约束无法滚动？"
date:   2016-11-14
excerpt: "今天做项目，碰到一个需求，在一个Cell中，因为需要适配4‘，所以需要将原本的一行按钮换成UIScrollView，可以滑动。"
tag:
- scrollView
- autolayout
comments: true
---

### 前言

今天做项目，碰到一个需求，在一个Cell中，因为需要适配4‘，所以需要将原本的一行按钮换成UIScrollView，可以滑动。如图：

![屏幕快照 2016-11-14 上午10.31.11.png](http://ocigwe4cv.bkt.clouddn.com/scrollView-autolayout_1.png)


#### 开始

这还不简单，直接将原本的存放按钮的`UIView`改成`UIScrollView`不就好咯。不一会就改好了，然后运行；发现还是不能动啊！（一脸懵逼）。
怎么办？又仔细查看了一遍，发现没有什么地方有写错阿。
然后Google了下，发现好像是autolayout这货的问题，于是我就试着不用autolayout看看能不能滚动，直接用frame来设置位置和大小，结果果然可以滚动了！

![屏幕快照 2016-11-14 上午11.01.10.png](http://ocigwe4cv.bkt.clouddn.com/scrollView-autolayout_2.png)

#### 然后

但到底是autolayout哪里出错了，于是去查了下官方文档，看看有没有什么可以发现的。果然让我找到了，原因如下：

>  The UIScrollView class scrolls its content by changing the origin of its bounds. To make this work with Auto Layout, the top, left, bottom, and right edges within a scroll view now mean the edges of its content view.

![Snip20161114_1.png](http://ocigwe4cv.bkt.clouddn.com/scrollView-autolayout_3.png)

如上图所示，之前我们设置的四个约束（上下左加宽度），其实参照物不是`scrollView`,而应该是外面的`contentSize`,为什么呢？因为`scrollView`是会滚动的，所以如果参照的是`scrollView`的话，那么按钮的位置就不是确定的了，所以参照的其实是`contentSize`。换句话说其实，`scrollView`就是根据内部按钮的frame计算出`scrollView`的`contentSize`。（因为autolayout本质依然是frame）。

这个时候我们再看下我们之前设置的四个约束，我们设置了上下左加宽度约束，这个时候发现`scrollView`并没有足够的条件去求出`contentSize`，因为右边并没有确定，所以这个时候滚动并没有效果。这个时候我们只需要加上按钮的右边约束就可以有效的算出`contentSize`,而且可以不用去设置`scrollView`的`contentSize`属性了，因为已经可以根据约束推测出来。

#### 最后

苹果的官方文档是个好东西☺️
