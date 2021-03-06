---
layout: post
title:  "【Tips】Cell Subview内按钮点击无响应？"
date:   2017-09-24
excerpt: "今天写一个界面的时候，在Cell中添加了一个子view，该view里有几个星星按钮，但是发现点击星星的时候，并没有响应。"
tag:
- iOS
comments: true
---

## 前言

今天写一个界面的时候，在Cell中添加了一个子view，该view里有几个星星按钮，但是发现点击星星的时候，并没有响应。**为啥？？**

![屏幕快照 2017-09-24 00.48.55.png]({{site.url}}/assets/images/blog/tableviewCell_btn_click_1.png)

## 番外

不应该啊，所以这时回想下事件传递机制。
我们知道能接受事件的都是`UIResponder`的子类，`UIView`,`UIViewController`都是`UIResponder`的子类，所以默认的能接受各种事件（touch事件，press事件，加速计事件，各种手势）。
> 但是默认的`UIImageView`,`UILabel`的`isUserInteractionEnabled`属性是NO,所以如果想要在他两上增加手势，或者ImageView中有其他的控件需要接受事件响应，这时我们需要手动的设置`isUserInteractionEnabled`属性为YES。

`UIApplication`我们很熟悉，一个全局的单例对象，App内所有的事件都是交给它进行分发和处理的。`UIApplication`除了能够接收上述的事件，还会接收`UIControl`的action事件，除此之外还有接受一些系统的事件（内存警告，屏幕旋转等）。

### 用户事件（Touch，Press）

当App接收到一个事件的时候，`UIKit`会自动的找到那个接收这个事件的`UIResponder`,所谓的`first responder`。

> 也可以手动调用`becomeFirstResponder`方法成为`first responder`，textView调用此方法就会自动弹出`inputView`,这里的`inputView`也就是我们的键盘,同时也可以显示自定义的`inputView`，比如常见的表情键盘。

![屏幕快照 2017-09-24 22.32.24.png]({{site.url}}/assets/images/blog/tableviewCell_btn_click_2.png)

#### HitTest

touch, press事件通过`hitTest(_:with:) `方法找到`first responder`。

`hitTest(_:with:) ` 又会从最外层的 view 的 subview 开始调用 `point(inside:with:) ` 方法判断点击的 point 是否在某个 subview 中， 如果该subview不包含该 point，继续寻找下一个，直到找到一个 subview 包含 point；如果该 subview 还有 subview 就一直递归下去，直到找到一个 subview，这个subview 就是下面所谓的 `first responder`，否则就是最外层的 view 成为 `first responder`。

![tableviewCell_btn_click_4.png]({{site.url}}/assets/images/blog/tableviewCell_btn_click_4.png)

比如上图是一个画板，侧边的礼物是一个 collectionView， 这个时候要求在侧边也需要可以绘制，就需要在侧边的礼物View中重写 `hitTest(_:with:) ` 方法，判断如果不是点击了cell，那么就将点击事件传递给外面的 subview。这样当用户绘制的区域不是点击cell的时候，就可以正常绘制了。

{% highlight swift %}
override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    let hitView = super.hitTest(point, with: event)
    if hitView == collectionView {
        return nil;
    }
    return hitView
}
{% endhighlight %}

> 👆代码中当 hitView 是 collectionView 时，返回 nil， 这个时候会找另一个subview 包含 point；也可以返回其他 subview 来代替自己。

#### Responder Chain

 找到`first responder`后，`first responder`有三种选择,下面以touch事件为例说明，点击蓝色的View，此时`first responder`为蓝色的View：

- 蓝色的View实现了`touchesBegan:withEvent:`，同时调用`super`,让事件进行传递，让下一个响应者做一些事情；

- 蓝色的View实现了`touchesBegan:withEvent:`，不调用`super`,此时事件传递中断；

- 蓝色的View没有实现`touchesBegan:withEvent:`方法，此时会根据`Responder Chain`进行寻找下一个响应者，如图所示：

![屏幕快照 2017-09-24 22.56.47.png]({{site.url}}/assets/images/blog/tableviewCell_btn_click_3.png)

> 我们也可以重写 `UIResponder` 的 `next` 属性重新指定下一个响应者。

### UIControl Action

当用户点击一个绑定target／action的按钮后，一个点击事件会被发送给`UIApplication`,然后`UIApplication`会通知到按钮去执行这个action。

(更详细的流程可以参考[这里](https://blog.ibireme.com/2015/05/18/runloop/))。

>  当addTarget，我们传了nil，也就是没有target，`UIApplication`就会通知first responder去执行action;

如果`first responder`没有实现这个action，那么此时就会按照事件那样根据`Responder Chain`进行寻找下一个响应者。

所以我们可以通过如下方式隐藏键盘

{% highlight swift %}
[[UIApplication sharedApplication] sendAction:@selector(resignFirstResponder) to:nil from:nil forEvent:nil];
{% endhighlight %}


### 系统事件

系统事件会直接发送给`UIApplication`，然后会分发到`AppDelegate`进行处理。


## 开始

回归正题，找到我们按钮添加事件的代码：

{% highlight swift %}
 starOne.addTarget(self, action: #selector(clickStar(_:)), for: .touchDown)
 starTwo.addTarget(self, action: #selector(clickStar(_:)), for: .touchDown)
{% endhighlight %}

扯了那么多，Cell中的按钮不能响应点击事件，说明当我们点击星星的时候，`UIApplication`并没有找到按钮添加的target，也就是`self`(EvaluateView),说明cell中的`evaluateView`并没有正确的显示，才导致按钮的点击没有正常的响应。

于是找到Cell中添加`evaluateView`的地方，发现问题了，设置约束不完整，少了一个底部的😂，添加以后`evaluateView`，正常显示，按钮的点击事件也就正常了。

## 最后

本文的Demo下载地址[星星Cell](https://github.com/longjianjiang/BlogDemo/tree/master/TableviewCellSubviewClickDemo)， [事件响应](https://github.com/longjianjiang/BlogDemo/tree/master/EventHandlingDemo)

## 一个疑问

UIControl的addTargetForSelector方法，触发的事件。touches方法只会停留在button这一层。笔者尝试新建一个UIButton的子类，重写touches方法，内部调用super，但是事件并没有往上传递。不知道这里UIControl内部是如何进行屏蔽的。

## References

[https://www.jianshu.com/p/d8512dff2b3e](https://www.jianshu.com/p/d8512dff2b3e)

[https://medium.com/ios-os-x-development/understanding-cocoa-and-cocoa-touch-responder-chain-12fe558ebe97](https://medium.com/ios-os-x-development/understanding-cocoa-and-cocoa-touch-responder-chain-12fe558ebe97)

[http://smnh.me/hit-testing-in-ios/](http://smnh.me/hit-testing-in-ios/)
