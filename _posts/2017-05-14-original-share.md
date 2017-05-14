---
layout: post
title:  "【Tips】iOS系统自带分享"
date:   2017-05-14
excerpt: "开发中经常会用到分享功能，下面介绍下苹果自带的分享发方式，个人认为还是比较好用的。"

tag:
- share
comments: true
---


### 前言
开发中经常会用到分享功能，下面介绍下苹果自带的分享发方式，个人认为还是比较好用的。

### 开始
分享效果如下：

![WechatIMG1.jpeg](http://upload-images.jianshu.io/upload_images/2050942-902f82eaba14a561.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


代码参考如下：
```
let text = "This is a simple weather app but can allow you write something to share with others."
let appURL = URL(string: "http://www.longjianjiang.com/drizzling/")
let image = UIImage.init(named: "name")
            
	    let textToShare = [text,appURL!] as [Any] //可以文字、图片、链接组合

	    let activityViewController = UIActivityViewController(activityItems: textToShare, applicationActivities: nil)

	     self.present(activityViewController, animated: true, completion: nil)
 ```	    
 >注意在模拟器上显示的分享选项是不准确的，建议使用真机测试。
 
 经过测试：
 - 微博支持纯文字、纯图片、文字加网页链接和文字加图片；
 - 微信支持文字加网页链接和纯图片；

 ### 最后
 喜欢原生的UI效果😂
