---
layout: post
title:  "【iOS动画】学习笔记第三弹(ViewController Transition Animation)"
date:   2017-10-14
excerpt: "本文是笔者学习ViewController Transition Animation的一些小总结,接第二弹"
tag:
- iOS
comments: true
---

>  本文是笔者学习iOS动画的一些小总结，接[第二弹](http://www.longjianjiang.com/animation-second/)；

## ViewController Transition Animation

之前在[第一弹](http://www.longjianjiang.com/animation-first/)和[第二弹](http://www.longjianjiang.com/animation-second/)中我们为某个view或者layer的某个属性做动画， 移动，放缩，渐变等效果。此时我们利用之前的知识，可以通过UIKit提供的API来自定义VC之间的转场动画，例如Modal动画，Push动画，Pop动画。

### Modal 动画


```
 present(modalVC, animated: true, completion: nil)
```


我们经常会用上述方式显示一个控制器，默认的效果是从下往上的一个动画，但是系统允许我们进行自定义这个呈现的效果，步骤如下：

1. 遵守`UIViewControllerTransitioningDelegate`协议

```
        let modalVC = ModalViewController()
        modalVC.transitioningDelegate = self
        present(modalVC, animated: true, completion: nil)
```

2. 实现两个协议的方法

```
extension ViewController: UIViewControllerTransitioningDelegate {
    func animationController(forPresented presented: UIViewController, presenting: UIViewController, source: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        return transition
    }
    
    func animationController(forDismissed dismissed: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        return transition
    }
}
```

这里如果返回`nil`，那么UIKit还是会用系统默认的动画效果进行呈现。

3. 实现 `UIViewControllerAnimatedTransitioning`两个方法
第二步中返回的对象是遵守`UIViewControllerAnimatedTransitioning`协议的对象；该协议有两个必须实现的方法：
` transitionDuration(using:)`: 告诉UIKit自定义的动画的时间；
`animateTransition(using:)`: 实现你想要的动画效果；

下面给出一个渐变的Modal和Dismiss动画栗子：

```
 func transitionDuration(using transitionContext: UIViewControllerContextTransitioning?) -> TimeInterval {
        return animationDuration
    }
    
    func animateTransition(using transitionContext: UIViewControllerContextTransitioning) {
        let containerView = transitionContext.containerView
        let toView = transitionContext.view(forKey: .to)
        
        containerView.addSubview(toView!)
        toView?.alpha = 0.0
        
        UIView.animate(withDuration: animationDuration, animations: {
            toView?.alpha = 1.0
        }, completion: { _ in
            transitionContext.completeTransition(true)
        })
    }
```


![屏幕快照 2017-10-13 10.21.27.png]({{site.url}}/assets/images/blog/animation_third_1.png)

> `transitionContext`在实现动画中非常重要，因为它提供了你需要做动画的对象，以Modal为例，默认的控制器的view叫做fromView，modal出来的控制器的view叫做toView，fromView和toView都是放到containerView中显示的，当动画结束后，fromView会从containerView中移除。

[VCModalAnimationDemo](https://github.com/longjianjiang/BlogDemo/tree/master/VCModalAnimationDemo)只是简单的一个效果，实际需要中，我们完全可以做出更加酷炫的转场动画。

### Push/Pop 动画
```
navigationController?.pushViewController(DetailViewController(), animated: true)
```
导航控制器进行Push操作的时候，默认的UIKit会提供一个从右往左的动画，同样的UIKit也允许我们进行自定义这个呈现效果，步骤如下：

1. 遵守`UINavigationControllerDelegate`协议
```
 navigationController?.delegate = self
```

2. 实现协议方法
```
 func navigationController(_ navigationController: UINavigationController, animationControllerFor operation: UINavigationControllerOperation, from fromVC: UIViewController, to toVC: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        transition.operation = operation
        return transition
    }
```
这里如果返回nil，那么UIKit还是会用系统默认的动画效果进行呈现。

3. 实现 UIViewControllerAnimatedTransitioning两个方法,这里其实和Modal动画是一样的，同样的是协议中两个必须要实现的方法。

下面给出一个Push时候进行放大图形进行展示新控制器View的动画，Pop时候进行放缩当前View直到消失的动画，效果如下图：

![屏幕快照 2017-10-13 14.31.16.png]({{site.url}}/assets/images/blog/animation_third_2.png)
![屏幕快照 2017-10-13 14.27.25.png]({{site.url}}/assets/images/blog/animation_third_3.png)

完整的Demo下载地址[VCPushPopAnimationDemo](https://github.com/longjianjiang/BlogDemo/tree/master/VCPushPopAnimationDemo)


## ViewController Interactive Transition Animation
我们不但可以自定义控制器之间的转场动画，我们还可以通过添加手势来控制转场动画，就像iOS7中新加的一个手势返回控制器的效果,步骤如下：

1. 实现`UINavigationControllerDelegate`协议方法

```
 func navigationController(_ navigationController: UINavigationController, interactionControllerFor animationController: UIViewControllerAnimatedTransitioning) -> UIViewControllerInteractiveTransitioning? {
        return transition.transitionInProgress ? transition : nil
    }
```
默认返回nil则不能控制动画；

上述代码中返回的`transition`对象是`UIPercentDrivenInteractiveTransition`的子类，该类遵守了`UIViewControllerInteractiveTransitioning`协议，使用该类我们不需要进行每一帧每一帧的控制动画，只需要传入根据手势移动的位移即可，该类会自动显示。

```
 @objc func handlePanGesture(recognizer: UIPanGestureRecognizer) {
        let translation = recognizer.translation(in: recognizer.view?.superview) // nivigation's view
        var progress: CGFloat = abs(translation.x / 200.0)
        progress = min(max(progress, 0.01), 0.99)
        
        switch recognizer.state {
        case .began:
           transitionInProgress = true
           navigationController.popViewController(animated: true)
        case .changed:
            update(progress)
        case .cancelled, .ended:
            if progress < 0.5 {
                cancel()
            } else {
                finish()
            }
            transitionInProgress = false
        default:
            break
        }
    }
```
完整的Demo下载地址[VCInteractiveNavigationAnimationDemo](https://github.com/longjianjiang/BlogDemo/tree/master/VCInteractiveNavigationAnimation)

## 最后
未完待续
