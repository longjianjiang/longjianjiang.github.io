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

实际GPU主要的耗时在于要进行Compositing，也就是将多个Texture进行拼接，也就是View的层级，`addSubview:`。第二个问题是图片尺寸，假设内存中有一张200*200的图片需要放到100*100的imageView中，如果不做处理，GPU需要对大图进行缩放显示，对像素点进行sampling，同时兼顾pixel alignment。第三个问题就是所谓的离屏渲染，主要问题在于渲染这种layer的时候，需要额外开辟内存，绘制好圆角这些，然后将绘制好的bitmap进行渲染，CALayer提供了一种缓存机制，`shouldRasterize`。


# AsyncDisplayKit


# References

[https://developer.apple.com/videos/play/wwdc2012/211/](https://developer.apple.com/videos/play/wwdc2012/211/)

[https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)

[http://foggry.com/blog/2015/05/06/chi-ping-xuan-ran-xue-xi-bi-ji/](http://foggry.com/blog/2015/05/06/chi-ping-xuan-ran-xue-xi-bi-ji/)

[https://www.zhangjiezhi.com/2018/12/13/Offscreen-rendering/](https://www.zhangjiezhi.com/2018/12/13/Offscreen-rendering/)
