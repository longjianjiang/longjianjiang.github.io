---
layout: post
title:  "【iOS动画】学习笔记第二弹(Layer Animation)"
date:   2017-10-08
excerpt: "本文是笔者学习Layer Animation的一些小总结,接第一弹"
tag:
- iOS
comments: true
---

> 本文是笔者学习iOS动画的一些小总结，接[第一弹](http://www.longjianjiang.com/animation-first/)；

## Layer Animation

[第一弹](http://www.longjianjiang.com/animation-first/)中主要是关于View Animation 的一系列操作，今天的主角当然得是Layer啦，其实Layer Animation 工作并不复杂，我们只需要选择一个是`animatable`的属性，然后设置开始值、结束值、动画时间，之后系统就会让Core Animation去对应的layer渲染，产生动画效果。

`CALayer`相比UIView有更多的`animatable`的属性，所以使用Layer Animation可以更好的写出自己想要的动画效果，同时`CALayer`还有很多特定的子类，常见的如下所示：`CAShapeLayer`,  `CATextLayer`, `CAGradientLayer`, `CAReplicatorLayer`….. 这些不同的子类，又包含不同的`animatable`的属性，所以根据特定的子类，可以简单的写出很酷炫的动画。下面来看一个栗子：

{% highlight swift %}
 let springMoveLeftAnimation = CABasicAnimation(keyPath: "position.x")
 springMoveLeftAnimation.fromValue = -view.frame.width
 springMoveLeftAnimation.toValue = view.frame.width / 2.0
 springMoveLeftAnimation.duration = 0.7
 springMoveLeftAnimation.timingFunction = CAMediaTimingFunction(name: kCAMediaTimingFunctionEaseOut)
 colorView.layer.add(springMoveLeftAnimation, forKey: nil)
{% endhighlight %}

上述栗子创建了一个`springMoveLeftAnimation`，这个动画对象是可以加到任意多个layer上去的，所以使用Layer Animation 创建的动画是可以重复利用的。上述栗子中只是简单的将一个view从左移动到屏幕中间。

### 真正的动画

> 上栗中，我们在`colorView`的layer上增加了一个动画，其实我们看到的动画并不是真正作用在`colorView`，我们看到的只是一个所谓的`presentation layer `,当动画结束后`presentation layer `就会在屏幕中被移除，真正的`colorView`重新显示到屏幕上。

所以上栗中，`colorView`的`x`原本不是居中的，经过动画后，`colorView`的`x`依然不是居中的，会移动到原本的位置。

所以想要`colorView`的`x`保持动画后的模样，我们可以设置如下代码：

{% highlight swift %}
springMoveLeftAnimation.fillMode = kCAFillModeBoth
springMoveLeftAnimation.isRemovedOnCompletion = false
{% endhighlight %}

#####  fillMode

`fillMode`是`CAMediaTiming`协议中一个属性，用来控制动画序列的开始和结束的行为，默认是`kCAFillModeRemoved`,默认效果如下图所示：

![屏幕快照 2017-09-20 15.36.00.png]({{site.url}}/assets/images/blog/animation_second_01.png)

> 如果想要延时执行某个动画，可以设置`beginTime`属性

{% highlight swift %}
springMoveLeftAnimation.beginTime = CACurrentMediaTime() + 0.3 
/// 根据CACurrentMediaTime()取得动画执行的时间，然后我们增加了0.3秒的延时
{% endhighlight %}

- kCAFillModeBackwards

![屏幕快照 2017-09-20 15.43.24.png]({{site.url}}/assets/images/blog/animation_second_02.png)

如图所示，设置`fillMode`为`kCAFillModeBackwards`,不论是否设置延时，都会提前显示动画的第一帧。

- kCAFillModeForwards

![屏幕快照 2017-09-20 15.46.24.png]({{site.url}}/assets/images/blog/animation_second_03.png)
如图所示，设置`fillMode`为`kCAFillModeForwards`,当动画被移除之前会保留动画的最后一帧。

- kCAFillModeBoth

![屏幕快照 2017-09-20 15.49.05.png]({{site.url}}/assets/images/blog/animation_second_04.png)
如图所示，设置`fillMode`为`kCAFillModeBoth`,相当于是上面两个属性的结合。

当动画结束后会保留最后一帧，然后设置`isRemovedOnCompletion`为`false`，所以动画就不会被移除，这样`colorView`就能够保持动画后的模样。但是现在因为不是真正的`colorView`，所以就不能做任何操作了，当`colorView`为输入框时，此时因为是`presentation layer `，也就不能响应用户的输入。而且这样做也会有性能问题，所以不建议设置`isRemovedOnCompletion`为`false`。

此时还有一个有效的方法是，通常我们可以在`colorView`加入动画之前，直接更新`colorView`的`x`，这样在动画结束后，`colorView`的位置就保持和动画后一致。

### 控制动画

 >  [第一弹](http://www.longjianjiang.com/animation-first/)中我们通过`UIKit`帮我们封装的UIView语法的动画一旦创建运行，我们是不能暂停或者停止的。但是Layer Aniamtion提供了相关的API，让我们可以更近一步的去控制我们所创建的动画。

#####  animation delegate

我们可以设置`CAAnimation`的代理，通过代理提供的方法来控制动画。

{% highlight swift %}
 func animationDidStart(_ anim: CAAnimation)
 func animationDidStop(_ anim: CAAnimation, finished flag: Bool)
{% endhighlight %}

`CAAnimationDelegate`提供了上述两个代理方法，通过提供的`anim`参数从而可以控制动画，但是如果多个动画都设置了代理，这时如何区分不同的动画做不同的事情呢？

因为`CAAnimation` 和其子类是用OC写的，而且支持KVC，所以我们可以用` func setValue(_ value: Any?, forKey key: String)`方法来设置不同key给不同的动画，这样在代理中就能区别不同的动画了。

通过设置代理的方式我们只能控制刚开始和刚结束时期的动画，那么如何控制正在运行的动画呢，此时就需要用到` func add(_ anim: CAAnimation, forKey key: String?)`方法中的`key`参数，我们可以在该动画开始后通过设置的`key`参数来控制对应的动画了。

{% highlight swift %}
 open func removeAllAnimations()
 open func removeAnimation(forKey key: String)
{% endhighlight %}

我们可以通过以上两个方法来对操作正在运行的动画，哈哈，只是简单的移除😂
并不能动态的改变动画运行路径。

> 我们可以设置动画的`speed`属性控制动画速度，同时也可以通过设置layer的`speed`属性从而使layer上添加的所有动画设置速度，此时如果layer里某个动画自身也设置了速度，那么此时的速度会根据view的层级进行乘积；

### 组合动画

我们可以在layer上添加多个动画，如果想要控制不同动画之间的同步，此时需要用到`CAAnimationGroup`

{% highlight swift %}
// added animation group
let groupAnimation = CAAnimationGroup()
groupAnimation.timingFunction = CAMediaTimingFunction(name: kCAMediaTimingFunctionEaseIn)
groupAnimation.beginTime = CACurrentMediaTime() + 0.5
groupAnimation.duration = 0.5
groupAnimation.fillMode = kCAFillModeBackwards

let scaleDown = CABasicAnimation(keyPath: "transform.scale")
scaleDown.fromValue = 3.5
scaleDown.toValue = 1.0
let rotate = CABasicAnimation(keyPath: "transform.rotation")
rotate.fromValue = .pi / 4.0
rotate.toValue = 0.0
let fade = CABasicAnimation(keyPath: "opacity")
fade.fromValue = 0.0
fade.toValue = 1.0

groupAnimation.animations = [scaleDown, rotate, fade]
loginButton.layer.add(groupAnimation, forKey: nil)
{% endhighlight %}

> 基本的layer animation 差不多就是这些，可能不是面面俱到，但是其他的一些属性，可以通过查看头文件里的属性即可，下面学习下layer animation里的spring animation；


###  弹性动画

[第一弹](http://www.longjianjiang.com/animation-first/)中我们就已经接触过layer animation，但是UIKit帮我们封装了，并没有深入的了解其中的细节，所以在layer 层中的spring animation我们可以进一步的研究下spring animation 的细节。

首先我们想象下钟摆，当我们给它一个力，此时如果没有摩擦的话，那么钟摆就会永远的摆动，但是实际由于和空气之间产生摩擦，所以钟摆最后一定会停止。而且不同质量的钟摆从运动到停止的时间是不一样的，最后还和重力有关系，同样的钟摆在月球上的运动轨迹和地球是有很大的差距的。

其实钟摆来回摆动的这个效果就是spring animation 的效果。此时我们可以根据上述总结出如下属性是影响spring animation的：

```
1. damping: 摩擦相关, 默认是10.0
2. mass: 质量，默认是1.0
3. stiffness: 重力相关, 默认是100.0
4. initialVelocity: 初始速度，默认是0.0
```

上述的四个属性就是`CASpringAnimation`所提供的，其中`damping`,`initialVelocity`我们在UIKit提供的spring animation 中就已经使用过，UIKit会根据我们所给的`duration`来计算出其他两个属性的，从而产生对应的spring animation，所以有时候会感觉有点不真实。但我们现在可以使用`CASpringAnimation`所提供的四个属性来创建自己想要的同时更加真实的spring animation， 但是这种方式的缺点是，时间是不确定的，因为spring animation 从开始到停止的时间是根据你所提供的四个影响参数来计算出的。

> 所以我们设置spring animation的 `duration`时，需要使用spring animation的`settlingDuration`属性，这个时间就是根据你所提供的四个影响参数来计算出的；

{% highlight swift %}
let flash = CASpringAnimation(keyPath: "borderColor")
flash.damping = 7.0
flash.stiffness = 200.0
flash.fromValue = UIColor(red: 1.0, green: 0.27, blue: 0.0, alpha: 1.0).cgColor
flash.toValue = UIColor.white.cgColor
flash.duration = flash.settlingDuration
textField.layer.add(flash, forKey: nil)
{% endhighlight %}

上述代码创建了一个spring animation 改变文本输入框的`borderColor`，要想做出自己想要的spring 动画，只需要不断的调整上述的四个属性即可。

###  keyframe动画

UIKit 也提供了keyframe animation，但是和layer层的keyframe animation 相比是有区别的。UIKit的keyframe animation，是用来做动画的连接的，可以在animations中创建多个keyframe动画，这些动画可以是设置在不同的view的不同属性。

之前我们设置在layer上的basic animation， 我们都会设置`fromValue`和`toValue`,通过设置的`duration`，Core Animation会自动的根据设置的value在给定的时间内改变layer的某个属性，于是动画产生了，而layer的keyframe animation则提供了让我们自己控制动画某个属性的过程的功能：

{% highlight swift %}
    let flight = CAKeyframeAnimation(keyPath: "position")
    flight.duration = 12.0
    flight.values = [
      CGPoint(x: -50, y: 0.0),
      CGPoint(x: view.frame.width + 50.0, y: 160.0),
      CGPoint(x: -50.0, y: loginButton.center.y)]
        .map { NSValue(cgPoint: $0) }
    
    flight.keyTimes = [0.0, 0.5, 1.0]
    balloon.add(flight, forKey: nil)
{% endhighlight %}

如上代码所示，我们通过`values`,`keyTimes`，自己可以控制动画的运行轨迹。

> 当创建的layer animation的keyPath是结构体时，需要使用NSValue进行包装；


## Specialized Layers

>前面所说的Layer Animation，我们都是为`CALayer`的一些基本属性做动画的，但是如果想要做出一些其他的酷炫的动画，哪些基本的可能满足不了要求，好在`CALayer`有很多有用的子类，我们通过它们的属性做动画，效果就会大不一样。

### CAShapeLayer

`CAShapeLayer`通过你传入的`path`让`vector graphics`来画你所定义的的图形。下面看个栗子：

{% highlight swift %}
let squarePath = UIBezierPath(rect: bounds)
let animation = CABasicAnimation(keyPath: "path")
animation.duration = 0.25
animation.fromValue = circleLayer.path
animation.toValue = squarePath.cgPath
circleLayer.add(animation, forKey: nil)
circleLayer.path = squarePath.cgPath

maskLayer.add(animation, forKey: nil)
maskLayer.path = squarePath.cgPath
{% endhighlight %}

和之前的layer animation一样，同样的设置`fromValue` 和 `toValue`，不过这次animation 的keyPath为`CAShapeLayer`的`path`属性。

`CAShapeLayer`的`strokeEnd`属性同样是`animatable`，所以根据这个属性可以做出一个画的过程的动画：

{% highlight swift %}
let pathAnimation = CABasicAnimation(keyPath: "strokeEnd")
pathAnimation.duration = 10.0
pathAnimation.fromValue = 0.0
pathAnimation.toValue = 1.0
pathLayer?.add(pathAnimation, forKey: "strokeEnd")
{% endhighlight %}

完整的Demo下载地址[CAShapeLayerAnimationDemo](https://github.com/longjianjiang/BlogDemo/tree/master/CAShapeLayerAnimationDemo)

> `CAShapeLayer`还有一个功能就是设置圆角，只需要设置一个shapeLayer作为layer 的`mask`属性即可。


### CAGradientLayer

`CAGradientLayer`可以通过设置多种颜色画出新的渐变的效果,而且`CAGradientLayer`有四个属性是`animatable`的：

> startPoint, endPoint 为单元坐标系

```
colors: 渐变效果的颜色数组；
locations: 每种颜色的在渐变中的占比；
startPoint: 开始的点；
endPoint: 结束的点；
```

iOS之前的滑动进行解锁的动画就可以使用`CAGradientLayer`来实现：

![WechatIMG1.png]({{site.url}}/assets/images/blog/animation_second_05.png)

完整的Demo下载地址[CAGradientLayerAnimationDemo](https://github.com/longjianjiang/BlogDemo/tree/master/CAGradientLayerDemo)

### CAReplicatorLayer

`CAReplicatorLayer`通常用来生成重复layer的集合，这样比手动添加效率更高。但是`CAReplicatorLayer`可以轻松的改变每个克隆layer的属性，让他们和他们的父亲不一样。最后`CAReplicatorLayer`有一个特别的属性`instanceDelay`，当你设置该属性为0.1秒同时在原layer上加了一个动画，此时`CAReplicatorLayer`生成的第一份克隆会延迟0.1秒执行动画，第二份克隆则会延迟0.2秒，第三份克隆则会延迟0.3秒，以此类推。通过这个特性我们可以写出一些复杂的动画效果。以下是`CAReplicatorLayer`三个重要的属性：

```
instanceCount: 设置你想要的克隆个数；
instanceTransform: 设置每个克隆和上一个克隆的差距；
instanceDelay： 设置每个克隆和上一个克隆的动画延迟；
```


利用`CAReplicatorLayer`上述特性，我们可以发挥想象做出一些酷炫的动画，例如下面这个动画：

![WechatIMG3.jpeg]({{site.url}}/assets/images/blog/animation_second_06.jpeg)

完整的Demo下载地址[CAReplicatorLayerAnimationDemo](https://github.com/longjianjiang/BlogDemo/tree/master/CAReplicatorLayerDemo)


## 最后

未完待续[第三弹](http://www.longjianjiang.com/animation-third/)
