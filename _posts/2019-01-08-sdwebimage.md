---
layout: post
title:  "SDWebImage阅读笔记"
date:   2019-01-08
excerpt:  "本文是笔者阅读SDWebImage的笔记"
tag:
- SourceCode
comments: true
---

SDWebImage是一个异步图片加载库，下面笔者来分析下它的源代码(v4.4.3)。

## 接口

{% highlight objective_c %}
[cell.imageView sd_setImageWithURL:[NSURL URLWithString:@"http://example.com/image.jpg"]
                      placeholderImage:[UIImage imageNamed:@"placeholder"]];
{% endhighlight %}

这是我们平时用到最多的接口，而且基本上不需要我们再额外设置什么了，接口内部会帮我们做好。

上面的接口是 `UIImageView` `WebCache` 分类中的一个方法，除了上面看到的这种，还有很多可选的，将很多参数进行缺省，形成很多子方法，内部都去调用下面这个包含所有参数的方法。

`sd_setImageWithURL:placeholderImage:options:progress:completed:`

除了给 `UIImageView` 增加了 `WebCache` 分类，还给 `UIButton`, `MKAnnotationView` 增加了`WebCache` 分类，接口定义方式和 `UIImageView` 一致，下面笔者以 `UIImageView` 为例进行叙述。

之前说到因为 `WebCache` 分类添加到多个UI组件中，所以此时给基类 `UIView` 也增加了 `WebCache` 分类，子类中的 `sd_setImageWithURL` 方法都会去调用父类中提供的实现方法，方法如下:

`sd_internalSetImageWithURL:placeholderImage:options:operationKey:setImageBlock:progress:completed:context:`

同时给 `UIView` 额外添加了一个 `WebCacheOperation` 分类，该分类主要是为了提供取消当前控件的图片加载, 方法如下:

`- (void)sd_cancelImageLoadOperationWithKey:(nullable NSString *)key;`

`sd_internalSetImageWithURL` 具体实现笔者将在下载和缓存后进行叙述。

## 下载

## 缓存

## 接口实现
