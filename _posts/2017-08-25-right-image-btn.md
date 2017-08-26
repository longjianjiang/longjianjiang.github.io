---
layout: post
title:  "【Tips】UIButton imageView 右边显示"
date:   2017-08-25
excerpt: "UIButton 默认图片显示在左边，但实际更多的将图片放在文字的左侧，比如--Follow➡️---"
tag:
- Tips 
comments: true
---

## 前言
UIButton默认图片显示在左边，但实际更多的是将图片放在文字的左侧，比如下图所示：

![屏幕快照 2017-08-25 22.27.45.png](http://ocigwe4cv.bkt.clouddn.com/rightImageBtn_1.png)

## 开始
所以得改造下UIButton，下面给出几种方法：

### transform
```
 rightImageBtn.transform = CGAffineTransform(scaleX: -1.0, y: 1.0) // 
将按钮文字和图片位置进行调换位置（图片并不会倒置）
 rightImageBtn.titleLabel?.transform = CGAffineTransform(scaleX: -1.0, y: 1.0) //将倒转的文字再次倒置（显示正常）
 rightImageBtn.imageEdgeInsets = UIEdgeInsets(top: 0, left: 0, bottom: 0, right: 10) //调整按钮内图片和文字之间的间隔
```
这个方式是笔者最推荐使用的，比较机智😂,从stackOverFlow上看到有人回答的；

### semanticContentAttribute
```
  rightImageBtn.semanticContentAttribute = .forceRightToLeft
  rightImageBtn.imageEdgeInsets = UIEdgeInsets(top: 0, left: 10, bottom: 0, right: 0)
```
这个方法同样也是比较机智，但是有个缺点就是仅支持`9.0`以上。这个苹果为了适配那种从右往左的语言而提出的，对于想要更改按钮的图片位置这个需求也是相当好用。

### edgeInset
```
 rightImageBtn.titleEdgeInsets = UIEdgeInsetsMake(0, -(rightImageBtn.imageView?.frame.size.width)!, 0, (rightImageBtn.imageView?.frame.size.width)!)
 rightImageBtn.imageEdgeInsets = UIEdgeInsetsMake(0, (rightImageBtn.titleLabel?.frame.size.width)! + 10, 0, -(rightImageBtn.titleLabel?.frame.size.width)!)
```
这个方法就比较常用了，设置`EdgeInsets`属性来达到效果。

### 自定义子类
```
 override func imageRect(forContentRect contentRect: CGRect) -> CGRect {
        var imageFrame = super.imageRect(forContentRect: contentRect)
        imageFrame.origin.x = super.titleRect(forContentRect: contentRect).maxX - imageFrame.width
        return imageFrame
    }
    
 override func titleRect(forContentRect contentRect: CGRect) -> CGRect {
        var titleFrame = super.titleRect(forContentRect: contentRect)
        if (self.currentImage != nil) {
            titleFrame.origin.x = super.imageRect(forContentRect: contentRect).minX
        }
        return titleFrame
    }

```
这个方法同样比较常用，重写上述两个方法，改变x坐标，从而达到效果。

## 最后
`transform`大法厉害😂

