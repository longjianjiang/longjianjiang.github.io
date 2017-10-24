---
layout: post
title:  "【Tips】Range & NSRange"
date:   2016-11-14
excerpt: "分析下Range和NSRange的不同"
tag:
- Swift
comments: true
---

### 前言

做个记录，比较下两者。

### 开始

NSRange OC中用来表示范围的，笔者第一次接触到是在截取字符串中：

```
 let nsStr: NSString = "com.longjianjiang"
 let nsNameRange = nsStr.range(of: "com")
```

上述结果为{0, 3}， 0为截取字符串的位置，3为截取字符串的长度，信息很直观；

但是笔者接触到Swift后，第一次使用的Swift中的Range要算是for循环中：

```
for i in 0..<5 {
     print(i)
 }
```

而且Swift中把很多之前OC中用NSRange来表示范围的都替换成了新的Range类型，比如同样的字符串截取：

```
 let str = "com😂.longjianjiang"
 let nameRange = str.range(of: "com😂")
```

上述结果为Index(offset: 0)..<Index(offset: 20), 是一个开区间，上述offset指的的是作为Unicode的字符偏移；

> 用NSRange来进行截取字符串时，如果字符串中含有emoji，此时就会少截，因为emoji比普通字符多占一个

```
        let myNSRange = NSRange(location: 3, length: 3)
        
        let myNSString: NSString = "longjianjiang"
        print(myNSString.substring(with: myNSRange)) // "gji"
        
        let myNSString2: NSString = "com😂.longjianjiang"
        print(myNSString2.substring(with: myNSRange)) // "😂."  少了 l 
```


> Swift4中的String类的subString方法标记为过时，建议使用切片下标, 看来很推荐使用Range类型

但是在处理富文本的时候，方法中的范围参数依然是之前的NSRange（不知道未来会不会也替换为Range）所以这时就涉及了如何两个不同范围类型的转换,不过很简单，系统有提供方法；

```
  let nsRange = NSRange(nameRange!, in: str) // str 为nameRange所在的字符串
  let range = Range.init(nsRange)
```

对比两者，其实我们可以发现新的Range类型所表达的东西比NSRange更多，用途也更加广泛，不仅仅可以表示范围，而且也可以用在循环中，所以这就引出了Range的不同类型：

其实根据上述我们知道不同Range有两种类型，开闭区间的Range、是否可迭代的Range，组合一下就是四种。可迭代的Range的元素得遵守Strideable协议，该协议说明元素是连续的同时支持偏移计算，该协议继承自Comparable，而且默认的Range的元素就得遵守Comparable协议。

#### 最后

有待补充
