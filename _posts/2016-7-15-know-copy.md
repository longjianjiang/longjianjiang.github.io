---
layout: post
title:  "认识copy关键字"
date:   2016-07-15
excerpt: "首先先引用阳神Sunny博客中的一道面试题：
用@property声明的NSString（或NSArray，NSDictionary）经常使用copy关键字，为什么？如果改用strong关键字，可能造成什么问题？
这说明对于我们来讲，弄懂copy还是十分有必要的，下面就让我们来一起看看copy的黑魔法。"
tag:
- copy
- 内存管理
- block
- property
comments: true
---

>首先先引用阳神Sunny[博客](http://blog.sunnyxx.com/2015/07/04/ios-interview/)中的一道面试题：
用@property声明的`NSString`（或`NSArray`，`NSDictionary`）经常使用`copy`关键字，为什么？如果改用`strong`关键字，可能造成什么问题？
>这说明对于我们来讲，弄懂`copy`还是十分有必要的，下面就让我们来一起看看`copy`的黑魔法。

* * *


# copy是什么，有什么用？

## 1.是什么？
首先`copy`和`mutableCopy`是方法，是`NSObject`内定义的方法。还有对应的类方法`copyWithZone:(struct _NSZone *)zone`以及两个协议`NSCopying`和`NSMutableCopying`
![Snip20160715_1.png](http://upload-images.jianshu.io/upload_images/2050942-b998b949e0decf7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Snip20160715_2.png](http://upload-images.jianshu.io/upload_images/2050942-124f0ffc28569a8b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 其中，`+copy:`、`+copyWithZone:`简单地说，是为了让"类"对象也符合NSCopying协议,也可以作为key插入`NSDictionary`中。又因为类对象全局只能存在一份，所以`+copy:`、`+copyWithZone:`方法只是简单返回self，而且这两个方法在ARC环境下也是不可用的。
- 对于`-copy`、`-mutableCopy`，这两个方法被调用就会产生一个新的副本对象，里面会直接把`-copyWithZone:`、`-mutableCopyWithZone:`的值返回。但是NSObject并没有实现`-copyWithZone:`和`-mutableCopyWithZone:`，所以子类对象要使用`-copy`、`-mutableCopy`就必须去实现`NSCopying`和`NSMutableCopying`协议。不过常见的`NSString`、`NSArray`、`NSDictionary`等都已遵守了上面两个协议。

- 最后`NSZone`已经被Apple抛弃，可不去追究。

## 2.有什么用？
copy顾名思义就是拷贝或者说克隆，所以copy的目的就是复制一份原来的内容，进一步思考为什么需要拷贝？显然：拷贝的目的就是***改变原来的内容不影响副本，改变副本也不影响原来的内容***

## 深拷贝、浅拷贝
下面通过`NSString`、`NSMutableString`和`NSArray`、`NSMutableArray`举例说明下上述两种拷贝是什么意思。

## 1.`NSString`、`NSMutableString`非容器对象分析
- 第一种情况：
![Snip20160715_4.png](http://upload-images.jianshu.io/upload_images/2050942-5d021b4fb6d2bb13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>我们注意到str和通过`[str mutableCopy]`的copyStr两者内容一致，但内容地址不同，也就是重新创建了一个对象。为什么要新建一个对象？
>1>：拷贝的目的是互不干扰，所以需要生成一个新的对象。
>2>：str是一个不可变的对象, 而通过`mutableCopy`拷贝出来的对象必须是一个可变的对象, 所以生成一个新的对象

- 第二种情况：

![Snip20160715_5.png](http://upload-images.jianshu.io/upload_images/2050942-98c004ad7a5a4f09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>我们发现copyStr在通过`[str mutableCopy]`之后，并没有因为str的改变而改变，符合拷贝的目的；同时这种情况拷贝也生成了一个新的对象，原因同上。

- 第三种情况：

![Snip20160715_6.png](http://upload-images.jianshu.io/upload_images/2050942-bbd09d94f1051153.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 此时copyStr在通过`[str copy]`之后，也没有因为str的改变而改变，同样也生成了一个新的对象，为什么？
>1>：拷贝的目的是互不干扰，所以需要生成一个新的对象。因为str是可变的，为了防止str改变后影响copyStr的值，所以必须新建对象。

- 第四种情况：

![Snip20160715_7.png](http://upload-images.jianshu.io/upload_images/2050942-f52e95869a388bbe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>此时我们发现并没有新建对象，这又是为什么呢？
>1>通过不可变对象调用了copy方法, 那么不会生成一个新的对象
>2> 因为原来的对象是不能修改的, 拷贝出来的对象也是不能修改的,既然两个都不能修改, 所以永远不能影响到另外一个对象,已经符合拷贝的目的 。所以，OC为了对内存进行优化, 就不会生成一个新的对象

## 2.`NSArray`、`NSMutableArray`容器对象分析
首先容器对象和非容器对象一样同样遵从下面的总结：

>如果对一不可变对象复制，copy是指针复制（浅拷贝）、mutableCopy就是对象复制（深拷贝）。
>如果是对可变对象复制，都是深拷贝，但是copy返回的对象是不可变的。

***
但是对于容器对象有两点特殊的地方：

- (1).`NSMutableArray`和`NSArray`多次copy有差别，请看图：


![Snip20160717_7.png](http://upload-images.jianshu.io/upload_images/2050942-3e0f2aa26429212d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

***

![Snip20160717_12.png](http://upload-images.jianshu.io/upload_images/2050942-e2aa848b1cace317.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也就是说：`NSMutableArray`多次copy每次都会新建对象而`NSArray`多次copy只新建一次对象。


- (2)对于容器而言，其元素对象始终是指针复制。这样我们就可以修改一个容器的值从而影响到其他拷贝的容器。

![Snip20160717_13.png](http://upload-images.jianshu.io/upload_images/2050942-95a3fd612c2447ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

***

如何实现元素对象也是对象复制？可以用归档的方法实现了真正的元素对象拷贝。

![Snip20160717_14.png](http://upload-images.jianshu.io/upload_images/2050942-62d16965b8ce80cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 3.总结
 正是因为调用copy方法有时候会生成一个新的对象, 有时候不会生成一个新的对象所以:
 >如果没有生成新的对象, 我们称之为浅拷贝, 本质就是指针拷贝
 >如果生成了新的对象, 我们称之为深拷贝, 本质就是会创建一个新的对象
最后：最重要的还是记住拷贝的目的，这样理解深浅拷贝都会变得非常简单，***改变原来的内容不影响副本，改变副本也不影响原来的内容***

## copy内存管理
>MRC下
>如果是浅拷贝:不会生成新的对象,但是系统就会对原来的对象进行retain。
>如果是深拷贝:会生成新的对象,系统不会对原来的对象进行retain。

## copy和property
- @property (nonatomic, copy) NSString *name;
这里为什么用copy，原因依然是拷贝的目的***改变原来的内容不影响副本，改变副本也不影响原来的内容***，因为如果不使用copy，别人修改了外界的name属性也会受影响，也就没有达到拷贝的目的。
- @property (nonatomic, copy) myBlock pBlock;block作为属性的时候也用copy，原因是：
 - 首先这涉及到MRC时代。因为MRC时期，为了防止block内用到的变量提前释放导致程序崩溃，使用copy将block存放到堆中，此时block会对内部变量进行一次retain操作，从而防止意外清空。同时block放入堆中也会带来一个新的问题，self持有block的引用，如果在block中使用self就会产生循环引用，所以不论MRC还是ARC，我们都分别用__blcok和__weak来修饰self。
 
 - 下面分别比较下MRC中retain和ARC中strong和copy的区别，看看strong是否也能达到同样的效果。
 
* * *

- 1.MRC下用retain修饰block.

![Snip20160716_4.png](http://upload-images.jianshu.io/upload_images/2050942-ca0d8ece6738f01c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们清楚的看到当点击屏幕调用blcok时，block内部用到的对象的提前释放，导致程序崩溃，而设置property属性的时候，Xcode也给了提示用copy替换retain。

- 2.ARC下用strong修饰block.

![Snip20160716_5.png](http://upload-images.jianshu.io/upload_images/2050942-bd2ee190ef051d29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
此时我们发现ARC下strong得效果和copy是一样的，同样可以防止内容对象被提前释放。

***

***所以结论copy是MRC下的产物，今天ARC时代为什么block依然使用copy，我想更多的是一种习惯问题***


## 尾巴
今天想用电脑通过屏幕共享 远程控制其他Mac 操作的，试了好几次，都没成功，最后还是QQ的远程好用，一键式的，以后做功能就得跟QQ学。
