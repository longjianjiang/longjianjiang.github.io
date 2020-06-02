---
layout: post
title:  "【iOS动画】学习笔记第一弹(View Animation)"
date:   2017-08-21
excerpt: "动画其实就是UIView基本属性(animatable)的操作，我们写动画的时候，其实不需要关心其中的数学计算，只需要熟悉API的特性即可。"
tag:
- iOS
comments: true
---

> 本文是笔者学习iOS动画的一些小总结；

## View Animation

动画其实就是`UIView`基本属性(`animatable`)的操作，我们写动画的时候，其实不需要关心其中的数学计算，只需要熟悉API的特性即可。

![屏幕快照 2017-08-20 22.45.54.png]({{site.url}}/assets/images/blog/animation_first_1.png)

我们可以看到`UIView`中的属性中注明为`animatable`的，那么这个属性就是我们写动画可以用到的属性。还有一个`alpha`属性同样是`animatable`。

本文中的动画方法都是`UIKit`为我们封装好的一些方法，也就是`UIView`提供的类方法，比如常见的`animate(withDuration:animations:)`,因为是`UIKit`层提供的，所以这些动画的方法都相对简单好用，没有涉及到 `Core Animation`提供的复杂API。

## animate

`UIView`的一个extension里都是`animate`的类方法，如下图所示：

![屏幕快照 2017-08-20 23.53.18.png]({{site.url}}/assets/images/blog/animation_first_2.png)

### normal

`UIView`的`animate`类方法是最简单的动画方法，相信大家一定都用过。

{% highlight swift %}
UIView.animate(withDuration: 0.5, delay: 0.4,
  options: [.repeat, .autoreverse, .curveEaseIn],
  animations: {
    self.password.center.x += self.view.bounds.width
  },
  completion: nil
)
{% endhighlight %}

> 这里想了好久不知道如何去解释，索性就不去解释了，反正iOSer能看懂就行😂

这里的`options`倒可以解释下, 这个属性可以告诉`UIKit`如何去执行`animations`里的操作,是`UIViewAnimationOptions`类型。常见的有

```
repeat // 重复的执行`animations`里的操作
curveEaseInOut //类似汽车行驶先加速之后到达目的地时减速的效果执行动画
curveEaseIn // 汽车加速
curveEaseOut // 汽车减速
curveLinear // 汽车匀速
```

### spring

spring类型的animate类方法相对于normal类型的增加了弹簧的效果，使动画更加符合现实中的动画效果。

{% highlight swift %}
 UIView.animate(withDuration: 1.5, delay: 0.0, usingSpringWithDamping: 0.2,
      initialSpringVelocity: 0.0, options: [],
      animations: {
        self.loginButton.bounds.size.width += 80.0
      },
      completion: nil
)
{% endhighlight %}

方法中`usingSpringWithDamping`取值为0-1，数值越大，弹簧效果越不明显。

### transition

之前的normal和spring都是基于`UIView`中`animatable`属性的动画，如果想要给添加view或者移除view做动画的话，就需要使用transition类型的animate类方法来实现。

{% highlight swift %}
UIView.transition(with: animationContainerView,
    duration: 0.33,
    options: [.curveEaseOut, .transitionFlipFromBottom],
    animations: {
      self.animationContainerView.addSubview(newView)
    },
    completion: nil
  )
{% endhighlight %}

上述代码给animationContainerView制造了一个动画，当有subview添加到animationContainerView中时，动画就会执行。方法中的options同样的告诉`UIKit`如何去执行`animations`里的操作,是`UIViewAnimationOptions`类型。常见的有

```
.transitionFlipFromLeft
.transitionFlipFromRight
.transitionCurlUp
.transitionCurlDown
.transitionCrossDissolve
.transitionFlipFromTop
.transitionFlipFromBottom
```

当我们想替换view时，可以使用下面的方法

{% highlight swift %}
UIView.transition(from: oldView, to: newView, duration: 0.33,
  options: .transitionFlipFromTop, completion: nil)
{% endhighlight %}

> UIKit帮了我们太多😂

### keyframe

之前的三种类型的动画都是单一效果的，现实中的动画大多是有许多步骤的，此时如果我们使用completion来连接下一个动画，代码会被嵌套很多层。所以我们可以将每个单独的步骤拆分成一个个的Keyframes，然后用Keyframe动画将各个单独的Keyframes连接起来生成一个完整的动画。

{% highlight swift %}
 UIView.animateKeyframes(withDuration: 1.5, delay: 0.0, animations: { 
            UIView.addKeyframe(withRelativeStartTime: 0.0, relativeDuration: 0.25, animations: { 
                self.planeImage.center.x += 80.0
                self.planeImage.center.y -= 10.0
            })
            
            UIView.addKeyframe(withRelativeStartTime: 0.1, relativeDuration: 0.4, animations: { 
                self.planeImage.transform = CGAffineTransform(rotationAngle: -.pi / 8)
            })
            
            UIView.addKeyframe(withRelativeStartTime: 0.25, relativeDuration: 0.25, animations: { 
                self.planeImage.center.x += 100.0
                self.planeImage.center.y -= 50
                self.planeImage.alpha = 0.0
            })
            
            UIView.addKeyframe(withRelativeStartTime: 0.51, relativeDuration: 0.01, animations: { 
                self.planeImage.transform = .identity
                self.planeImage.center = CGPoint(x: 0.0, y: originalCenter.y)
            })
            
            UIView.addKeyframe(withRelativeStartTime: 0.55, relativeDuration: 0.45, animations: { 
                self.planeImage.alpha = 1.0
                self.planeImage.center = originalCenter
            })
            
        }, completion: nil)
{% endhighlight %}

上例写了一个飞机起飞到着陆的动画，`addKeyframe`里的`withRelativeStartTime`和`relativeDuration`都是相对于`animateKeyframes`中`withDuration`来计算的。

## auxiliary views

{% highlight swift %}
 // cube animation
    func cubeTransition(label: UILabel, text: String, direction: AnimationDirection) {
        let auxLabel = UILabel(frame: label.frame)
        auxLabel.text = text
        auxLabel.font = label.font
        auxLabel.textAlignment = label.textAlignment
        auxLabel.textColor = label.textColor
        auxLabel.backgroundColor = label.backgroundColor
        
        let auxLabelOffset = CGFloat(direction.rawValue) * label.frame.size.height / 2.0
        auxLabel.transform = CGAffineTransform(scaleX: 1.0, y: 0.1)
         .concatenating(CGAffineTransform(translationX: 0.0, y: auxLabelOffset))
        label.superview?.addSubview(auxLabel)
        
        UIView.animate(withDuration: 0.5, delay: 0.0, options: [.curveEaseOut], animations: { 
            auxLabel.transform = .identity
            label.transform = CGAffineTransform(scaleX: 1.0, y: 0.1)
             .concatenating(
                CGAffineTransform(translationX: 0.0, y: -auxLabelOffset)
            )
        }, completion: { _ in
            label.text = auxLabel.text
            label.transform = .identity
            auxLabel.removeFromSuperview()
        })
    }
{% endhighlight %}

上述代码写了一个label立方体翻转的效果，但其实不是真的3D效果，这里使用到了一个辅助的label，对两个label同时动画，最后移除这个辅助label，从而达到了立方体翻转的效果。

## Auto Layout

之前的动画都是设置`animatable`的属性，UIKit帮助我们实现对应的动画效果。但是现在我们大多更多的使用Auto Layout来进行布局界面，此时上面的几种动画方法同样也是有效的，只是此时我们操作的是约束。

需要在animation block里调用父view的`layoutIfNeeded()`, 来进行更新约束。

# UIViewPropertyAnimator 更新于2017-11-11

UIViewPropertyAnimator 是iOS10新出的一个动画相关的类，通过这个类我们可以简单的创建更加**易于控制**的view animation，也就是说此时我们在一些场景中可以不去创建layer animation，通过UIViewPropertyAnimator也可以完成需要。

UIViewPropertyAnimator和之前的UIView.animate系列方法是相辅相成的，也就是说有些简单的情况，我们只需要使用UIView.animate系列方法就可以了，没有必要去使用UIViewPropertyAnimator。

一个简单的UIViewPropertyAnimator创建的动画如下：

{% highlight swift %}
let scale = UIViewPropertyAnimator(duration: 0.33, curve: .easeIn)
   // addAnimations 也可以添加之前的上文所说的keyFrame animation等
    scale.addAnimations {
      view.alpha = 1.0
    }
    scale.addAnimations({
      view.transform = CGAffineTransform.identity
    }, delayFactor: 0.33) // 这里的delayFactor是一个相对比例（0-1），是相对于动画剩下的时间，这样就保证了延迟的时间不会超过总时间
    scale.addCompletion {_ in
      print("ready")
    }
{% endhighlight %}

> 各种不同的animator 可以封装在单独的一个类，这样不同的view如果想要创建相同的动画直接调用方法然后start即可。

### animation timing

UIViewPropertyAnimator默认提供了以下的curve选择：

{% highlight swift %}
public enum UIViewAnimationCurve : Int {

    
    case easeInOut // slow at beginning and end

    case easeIn // slow at beginning

    case easeOut // slow at end

    case linear
}
{% endhighlight %}

其实这些curve都是控制两个control point形成的贝塞尔曲线，具体可以去这个网站实际体验下[curve](http://cubic-bezier.com/#.47,-0.32,.96,.88)；

UIViewPropertyAnimator提供了自定义这两个control point的方法，可以让我们自定义timing 效果。

{% highlight swift %}
  public convenience init(duration: TimeInterval, controlPoint1 point1: CGPoint, controlPoint2 point2: CGPoint, animations: (() -> Swift.Void)? = nil)
{% endhighlight %}

除了使用自定义control point的方式来自定义timing，我们还可以通过下面这个方法来为animator提供自定义的timing方式：

{% highlight swift %}
public init(duration: TimeInterval, timingParameters parameters: UITimingCurveProvider)
{% endhighlight %}

UITimingCurveProvider是一个协议，UIKit提供了两个已经遵守该协议的类，我们可以方便的拿来使用。
UICubicTimingParameters and UISpringTimingParameters.  例如下面这个spring timing：

{% highlight swift %}
 let spring = UISpringTimingParameters(dampingRatio: 0.55)
    
 let animator = UIViewPropertyAnimator(duration: 1.0, timingParameters: spring)
{% endhighlight %}

这样这个animator的动画运行时就是弹簧效果了。

### animation state

因为UIViewPropertyAnimator可以创建##易于控制##的动画，所以UIViewPropertyAnimator可以让我们知道现在动画的状态，以下是三个主要的状态属性：

```
1.isRunning(Bool): animator 的动画是否正在运行，默认是false，当调用`startAnimation`后变为true；
2.isReversed(Bool): animator 的动画执行顺序，默认是false，当设置为true后，动画会反方向执行，此时才改变没有效果，也就是只能一次性设置；
3.state: animator 动画的状态，具体见下图：
```

![屏幕快照 2017-11-11 16.57.24.png]({{site.url}}/assets/images/blog/animation_first_3.png)

> fractionComplete可以接受一个animator的完成百分比，达到一种可交互的动画，类似3Dtouch 的重按动画效果；

### ViewController Transition Animation

同样的正是因为UIViewPropertyAnimator可以让我们控制动画的一系列状态，我们也可以使用它来实现控制器的可交互切换动画，下面方法是iOS10中`UIViewControllerAnimatedTransitioning`协议中新增的一个方法，允许提供一个animator：

{% highlight swift %}
func interruptibleAnimator(using transitionContext:
  UIViewControllerContextTransitioning) -> UIViewImplicitlyAnimating
{% endhighlight %}


总的来说，UIViewPropertyAnimator是一个中间产物，为我们提供了多一种的选择，不及UIView.animate 来的简单，不及Layer Animation的全面。

## 最后

未完待续[第二弹](http://www.longjianjiang.com/animation-second/)
