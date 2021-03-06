---
layout: post
title:  "关于知识链"
date:   2017-10-25
excerpt: "本文是笔者的一个真实经历，想通过本文分享下自己对于学习的看法，不涉及技术，只是一点点想法；"
tag:
- Thought
comments: true
---

> 本文是笔者的一个真实经历，想通过本文分享下自己对于学习的看法，不涉及技术，只是一点点想法；

遇到这样一个场景，Label中显示的文本中一段字符串为邮箱，此时想要让邮箱字符串高亮显示同时添加点击事件。

这个时候笔者想到了为Label增加`分类`，分类比继承更加友好，侵入性不那么高。

因为分类中默认不能添加属性，实际中如果分类中的方法较复杂，那么肯定是会添加属性的，这个时候`Runtime`就派上用场了。

要想判断点击Label时，点击点是否在邮箱字符串上，也就是高亮区域，这时我们要做的就是计算出高亮区域的Rect，如何计算？
好像并没有提供好的方法，笔者经过查询知道了可以通过 `TextKit`来计算出高亮区域（在Label中的Rect）; 同样的笔者发现也可以通过`CoreText`来计算出高亮区域。

计算出高亮区域后，我们通过Label的touch方法可以方便的判断点击是否在高亮区域，然后就可以增加点击后的事件。

上述一个简单的例子，引出了一长串知识链，也就是说其实我们在日常开发中会不断的遇到类似引发知识链的情况，如果我们一步一步的按照知识链去学习，最后解决问题，对技术的提升有很大帮助。

```
category -> Runtime -> AttributeString -> TextKit -> CoreText 
```

但每个知识链都可以继续纵向的扩展，毕竟我们的首要任务还是解决问题，所以笔者认为在学习知识链中某个链知识时，可以适当的克制我们的好奇心，暂且不纵向扩展到其他的知识，先解决问题，但是可以先做个标记，以便未来方便继续。

以上就是笔者一点小小的学习体会。
