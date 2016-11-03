---
layout: post
title:  "【Tips】iOS开发中的汉字拼音排序"
date:   2016-11-03
excerpt: "今天做项目，碰到一个需求，将列表中的cell按照姓名排序显示，其实也简单，就是把数据源里的模型按照姓名重置顺序即可。"
tag:
- Tips
comments: true
---

### 前言
今天做项目，碰到一个需求，将列表中的cell按照姓名排序显示，其实也简单，就是把数据源里的模型按照姓名重置顺序即可。


#### 开始
这个之前还真没遇到过，于是找好朋友`Google`,果然给力，第一个结果就是我想要的。
![屏幕快照 2016-11-03 下午4.26.44.png](http://ocigwe4cv.bkt.clouddn.com/pinyinSort_1.png)

结果发现不但是我想要的，而且还简单，于是就写了个Demo，发现果然可以实现。

主要利用数组的`sortedArrayUsingFunction:context:`方法，传入一个自定义的比较方法，该方法利用`NSString`的`localizedCompare:`方法。
![屏幕快照 2016-11-03 下午4.31.47.png](http://ocigwe4cv.bkt.clouddn.com/pinyinSort_2.png)

但是我用同样的方法在项目中，发现并没有完全实现排序，顺序还是错乱，于是我新建了一个iOS平台下的Demo，发现果然是有问题的。
![屏幕快照 2016-11-03 下午4.35.34.png](http://ocigwe4cv.bkt.clouddn.com/pinyinSort_3.png)
同样的代码，换了个平台出问题了，不知怎么回事，如果有大神知道，还请指教。

#### 然后
想到可以先把汉字转成拼音形式的英文类型字符串，然后根据字母进行排序就好，于是发现系统自带方法就可以处理，就不需要第三方库了
>（PS:第三方转化的比较常用的是由George编写的，使用起来比较方便，这个库转化是将汉字转化成汉字的拼音首字母。有兴趣的同学可以自行搜索这个文件。）

代码如下所示：

```
 NSMutableString *mutableString = [NSMutableString stringWithString:model.person_name];
 CFStringTransform((CFMutableStringRef)mutableString, NULL, kCFStringTransformToLatin, false);
 CFStringTransform((CFMutableStringRef)mutableString, NULL, kCFStringTransformStripDiacritics, false);
 mutableString =(NSMutableString *)[mutableString stringByReplacingOccurrencesOfString:@" " withString:@""];
 
```

为此我在原先数据模型中新增一个字段用来保存汉字转拼音的值，接着按这个字段将模型数组的数据进行排序即可。

```
  NSArray *sortDescriptors = [NSArray arrayWithObject:[NSSortDescriptor sortDescriptorWithKey:@"person_name_pinyin" ascending:YES]];//person_name_pinyin为新增的保存拼音的字段
  [unSortedModelArray sortUsingDescriptors:sortDescriptors];

```

#### 最后
完成，拼音排序。
