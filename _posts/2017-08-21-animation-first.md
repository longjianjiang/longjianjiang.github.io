---
layout: post
title:  "【iOS动画】学习笔记第一弹"
date:   2017-08-21
excerpt: "动画其实就是UIView基本属性(animatable)的操作，我们写动画的时候，其实不需要关心其中的数学计算，只需要熟悉API的特性即可。"
tag:
- Animation
comments: true
---

> 本文是笔者学习iOS动画的一些小总结；

## View Animation
动画其实就是`UIView`基本属性(`animatable`)的操作，我们写动画的时候，其实不需要关心其中的数学计算，只需要熟悉API的特性即可。

![屏幕快照 2017-08-20 22.45.54.png](http://ocigwe4cv.bkt.clouddn.com/animation_first_1.png)

我们可以看到`UIView`中的属性中注明为`animatable`的，那么这个属性就是我们写动画可以用到的属性。还有一个`alpha`属性同样是`animatable`。

本文中的动画方法都是`UIKit`为我们封装好的一些方法，也就是`UIView`提供的类方法，比如常见的`animate(withDuration:animations:)`,因为是`UIKit`层提供的，所以这些动画的方法都相对简单好用，没有涉及到 `Core Animation`提供的复杂API。

## animate
`UIView`的一个extension里都是`animate`的类方法，如下图所示：

![屏幕快照 2017-08-20 23.53.18.png](http://ocigwe4cv.bkt.clouddn.com/animation_first_2.png)

### normal
`UIView`的`animate`类方法是最简单的动画方法，相信大家一定都用过。
```
UIView.animate(withDuration: 0.5, delay: 0.4,
  options: [.repeat, .autoreverse, .curveEaseIn],
  animations: {
    self.password.center.x += self.view.bounds.width
  },
  completion: nil
)
```
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
```
 UIView.animate(withDuration: 1.5, delay: 0.0, usingSpringWithDamping: 0.2,
      initialSpringVelocity: 0.0, options: [],
      animations: {
        self.loginButton.bounds.size.width += 80.0
      },
      completion: nil
)
```
方法中`usingSpringWithDamping`取值为0-1，数值越大，弹簧效果越不明显。

### transition
之前的normal和spring都是基于`UIView`中`animatable`属性的动画，如果想要给添加view或者移除view做动画的话，就需要使用transition类型的animate类方法来实现。

 ```
UIView.transition(with: animationContainerView,
    duration: 0.33,
    options: [.curveEaseOut, .transitionFlipFromBottom],
    animations: {
      self.animationContainerView.addSubview(newView)
    },
    completion: nil
  )
```
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
```
UIView.transition(from: oldView, to: newView, duration: 0.33,
  options: .transitionFlipFromTop, completion: nil)
```
> UIKit帮了我们太多😂

### keyframe
之前的三种类型的动画都是单一效果的，现实中的动画大多是有许多步骤的，此时如果我们使用completion来连接下一个动画，代码会被嵌套很多层。所以我们可以将每个单独的步骤拆分成一个个的Keyframes，然后用Keyframe动画将各个单独的Keyframes连接起来生成一个完整的动画。

```
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
```
上例写了一个飞机起飞到着陆的动画，`addKeyframe`里的`withRelativeStartTime`和`relativeDuration`都是相对于`animateKeyframes`中`withDuration`来计算的。

## auxiliary views
```
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
```
上述代码写了一个label立方体翻转的效果，但其实不是真的3D效果，这里使用到了一个辅助的label，对两个label同时动画，最后移除这个辅助label，从而达到了立方体翻转的效果。

## Auto Layout
之前的动画都是设置`animatable`的属性，UIKit帮助我们实现对应的动画效果。但是现在我们大多更多的使用Auto Layout来进行布局界面，此时上面的几种动画方法同样也是有效的，只是此时我们操作的是约束。

## 最后
未完待续[第二弹](http://www.longjianjiang.com/animation-second/)
