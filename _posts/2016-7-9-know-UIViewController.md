---
layout: post
title:  "认识UIViewController"
date:   2016-07-9
excerpt: "UIViewController在开发中每天都要打交道的，通常我们用它来管理页面的View，现在就系统的研究下我们常说的MVC中的C。"
tag:
- iOS

comments: true
---


UIViewController在开发中每天都要打交道的，通常我们用它来管理页面的View，现在就系统的研究下我们常说的`MVC`中的C。

## 创建UIViewController
三种方式：

- new

{% highlight objective_c %}
UIViewController *vc = [UIViewController new];
{% endhighlight %}

- 从Storyboard中创建

{% highlight objective_c %}
UIStoryboard *sb = [UIStoryboard storyboardWithName:@"xxx" bundle:nil];
UIViewController *vc = [sb instantiateInitialViewController];
{% endhighlight %}

- 从Xib中创建

{% highlight objective_c %}
UIViewController *vc = [[UIViewController alloc] initWithNibName:@"xxx" bundle:nil];
{% endhighlight %}

## 创建UIViewController的View
我们知道每个控制器都有一个页面也就是常写的`self.view`。先看一张草图：
![Snip20160709_1.png](http://ww4.sinaimg.cn/mw1024/6b7cdce2gw1f6uttnmvexj20mj0ih40i.jpg)
上图描述了控制器View加载的过程，下面给出几点建议

- 1.`loadView`方法用来自定义View，此时不论控制器从Storyboard中创建还是从Xib中加载View，统统失效，以`loadView`方法为准。

- 2.因为默认系统会去寻找特定名称的两个Xib，所以开发中我们如果要用到控制器View的Xib，起名字和控制器同名,避免使用图例中LJOneView，因为项目中很有可能存在LJOneView，这样会产生混淆。


## UIViewController生命周期方法
看图说话：
![生命周期方法.png](http://ww3.sinaimg.cn/mw1024/6b7cdce2gw1f6uttm5nyij20yg0dhtbp.jpg)
由图可以清楚的看到控制器View的由生到死，两点注意：

- 控制器的View是懒加载，所以只有当销毁View后才会重新创建View。

- `viewWillUnload`、`viewDidUnload`现在已被系统弃用。

## UIViewController层级关系
- 单控制器
根控制器决定了这个控制器的View是否可以被显示，此时只有一个控制器。
```
LJOneViewController *oneVC = [LJOneViewController new];
self.window.rootViewController = oneVC;
```

- 多控制器
 - 显然实际肯定不止一个控制器，比如最常见的`UINavigationController`是一种多控制器的组合，所以它显示的内容由子控制器的View和导航栏（有时还有工具条）组成。此时导航控制器的View始终在在Window的最上层。
 - 同时通过Modal方式也可以出现这种多控制器的情况，每次modal出来的控制器要显示的内容依然是放在源控制器的View上。如果有多次modal，那么最上层控制器的View内容显示在它下层的控制器View中，指导最底层控制器。

## UIViewController中View的理解及作用
- UIView通常用来显示内容，控制器的View也不例外，我们可以从Storyboard中拖一些控件进行布局。***控制器View换句话说也可以被看做一个容器，用来存放其他要显示内容的容器。***

![Snip20160709_2.png](http://ww4.sinaimg.cn/mw1024/6b7cdce2gw1f6uttmvuwdj20fd09emxl.jpg)
图中每个控制器强引用Root View，然后Root View对通过`[self.view addSubview:xxx];`的进行强引用，这也就是我们通过Storyboard拖线的控件默认是`weak`弱引用。
- 控制器的View也可以管理其他控制器的View，例如`UISplitViewController`

![Snip20160709_3.png](http://ww2.sinaimg.cn/mw1024/6b7cdce2gw1f6uttl3fzqj20jf0f53zd.jpg)
此时`UISplitViewController`的View管理着两个子控制器的View，而两个子控制器的View则管理对应的Root View的子View,最终两个子控制器View上的内容仍然放在`UISplitViewController`的View这个容器中显示。

## 尾巴
Sketch画图感觉飞起来了！
