---
layout: post
title: "UIView 渲染学习笔记"
date: 2019-07-10
excerpt: "nope"
tag:
- iOS
comments: true
---

# UIView绘制

iOS屏幕刷新率是每秒60帧，也就是16.7ms会绘制一次屏幕，这段时间会CPU会完成view缓冲区的创建和view内容的绘制，也就是`drawRect:`方法。内容绘制完成后，接下来的任务就是GPU进行渲染，GPU渲染的主要工作在于view的拼接，纹理的渲染，最终内容显示在屏幕上。

每个UIView有一个layer，每个CALayer都有一个contents，这个contents指向的是一块内存bitmap(backing store)，`UIGraphicsGetCurrentContext()`指向的其实就是backing store。当backing store中的内容写入完成后，通过render server另一个进程提交给GPU去渲染，将backing store中的bitmap显示在屏幕上。

UIView绘制的时候，CPU执行`drawRect`方法，通过contents将数据写入backing store，CPU的主要工作就是进行绘制，得到bitmap。

GPU处理的单位是Texture，控制GPU的是OpenGL，所以bitmap到Texture之间需要进行一次转换，Core Animation就是完成这层转换的。Core Animation会创建一个OpenGL的Texture和CGImageRef(bitmap)绑定，通过TextureId进行标识。绑定关系建立后，剩下的事情就是GPU将Texture渲染到屏幕上了。大致过程是这样的，GPU将内存中的bitmap搬到VRAM中处理，然后将Texture进行渲染。

实际GPU主要的耗时在于要进行Compositing，也就是将多个Texture进行拼接，也就是View的层级，`addSubview:`。第二个问题是图片尺寸，假设内存中有一张200*200的图片需要放到100*100的imageView中，如果不做处理，GPU需要对大图进行缩放显示，对像素点进行sampling，同时兼顾pixel alignment。第三个问题就是所谓的离屏渲染，主要问题在于渲染这种layer的时候，需要额外开辟内存，绘制好圆角这些，然后将绘制好的bitmap进行渲染，这中间会涉及到上下文切换，CALayer提供了一种缓存机制，`shouldRasterize`。

以上过程也是Runloop相关的，当一个Runloop开始的时候，会调用`[CATransaction begin]`，Runloop结束的时候调用`[CATransaction commit]`，begin和commit之间做一个layer tree的操作，比如修改view的层级，修改layer的属性，view的frame等等。commit之后，CPU进行绘制，GPU进行渲染，最终展示到屏幕中。

# AsyncDisplayKit

AsyncDisplayKit 是一个保持界面流程的库，我们已经大概了解了Display的流程，而这个库优化的点其实也是针对CPU和GPU两个方面。

CPU：从A5开始多核成为主流，所以可以将一些耗时的CPU操作放到后台线程，从而减轻主线程的工作，多核可以满足后台线程的并发执行。

GPU：减少Compositing，也就是将一些不需要响应事件的sub-layer合成渲染成一张Texture，这个工作同样可以放到后台线程。

AS通过创建Node封装了UIView和CALayer，使用Node就像使用View一样，Node是线程安全的，可以在后台线程创建，layout，绘制Node。

AS内部注册了一个Runloop的observer，监听了`beforeWaiting`，`exit`事件，所有Node的提交会放在`_ASAsyncTransactionGroup`中，等到一次Runloop到来的时候，一次性将group中的transaction进行commit。此时绘制已经在后台线程完毕，将绘制好的image赋值给layer的contents，通知Node显示完成。

# References

[https://developer.apple.com/videos/play/wwdc2012/211/](https://developer.apple.com/videos/play/wwdc2012/211/)

[https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)

[http://foggry.com/blog/2015/05/06/chi-ping-xuan-ran-xue-xi-bi-ji/](http://foggry.com/blog/2015/05/06/chi-ping-xuan-ran-xue-xi-bi-ji/)

[https://www.zhangjiezhi.com/2018/12/13/Offscreen-rendering/](https://www.zhangjiezhi.com/2018/12/13/Offscreen-rendering/)
