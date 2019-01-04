---
layout: post
title:  "【Tips】Swift Pointer"
date:   2017-11-1
excerpt:  "本文是一个Swift中指针的笔记"
tag:
- Swift
comments: true
---

### 前言

在Swift中使用sqlite3 这个库的时候，因为是C的库，所以少不了指针，但是一开始少不了一些困惑。

所以只能先查下相关的资料啦，下面是笔者的一个学习笔记：

### 开始

Swift中常见的用来表示指针的类型有：

```
UnsafePointer： const修饰的指针
UnsafeMutablePointer： 普通类型指针
UnsafeBufferPointer： 集合类型指针
OpaquePointer:  结构体指针, sqlite3 就是用这个类型的指针
UnsafeRawPointer: 相当于UnsafePointer<Void>, 也就是C语言中的Void *
```

有了以上的对应关系，在处理开发中C语言的库时，我们就能够知道各种Unsafe类型具体指的是哪一种，因为我们更多的使用库，并不需要去写，其实基本的就是这些类型熟悉就好。

笔者记得OC中有些方法是通过传入一个NSError指针(二级指针)返回错误的， 也有一些通过指针参数进行传递参数的：

{% highlight objective_c %}
- (BOOL)writeToURL:(NSURL *)url options:(NSDataWritingOptions)writeOptionsMask error:(NSError **)errorPtr;
{% endhighlight %}

以上方法Swift中可以通过传入UnsafeMutablePointer<NSError>  error类型即可，方法内部通过给error的pointee 属性赋一个定义好的NSError即可。

但是在Swift中，这种通过指针返回错误的方法被替换为可以抛出异常的方法，所以我们在设计的时候，最好还是按照Swift的方法去写，这样更加通用。

不过这种通过指针传递参数的，Swift中同样的是使用了UnsafeMutable类的指针进行传递的：

{% highlight swift %}
- (void)popoverPresentationController:(UIPopoverPresentationController *)popoverPresentationController willRepositionPopoverToRect:(inout CGRect *)rect inView:(inout UIView  * __nonnull * __nonnull)view;

optional public func popoverPresentationController(_ popoverPresentationController: UIPopoverPresentationController, willRepositionPopoverTo rect: UnsafeMutablePointer<CGRect>, in view: AutoreleasingUnsafeMutablePointer<UIView>)
{% endhighlight %}

> AutoreleasingUnsafeMutablePointer 这种类型的指针不引用对象，只存储对应的改变给指针的值

#### 最后

未完待续
