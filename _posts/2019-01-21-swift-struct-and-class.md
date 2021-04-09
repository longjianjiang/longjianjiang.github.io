---
layout: post
title:  "【Tips】Swift 中 struct 和 class 选谁？"
date:   2019-01-21
excerpt:  "本文介绍了Swift中新建对象选择struct还是class。"
tag:
- Swift
comments: true
---

> Swift 中有 class 和 struct，而且 struct 相比传统的C语言中的struct多出了很多功能，使swift中的struct更加像class，这就出现一个问题，怎么选择，当我们创建自定义对象的时候，选择哪一个？

首先给出一个[答案](https://www.mikeash.com/pyblog/friday-qa-2015-07-17-when-to-use-swift-structs-and-classes.html): 当需要值语义就选struct；当需要引用语义，就选class。

上面答案提到了值和引用，两者的一个区别其实是存储内容的地方不同。值类型的数据通常存放在栈上，而引用类型的数据通常则存放在堆上。

下面笔者拿Swift中的数组类型来说明值类型和引用类型的区别：

现在Swift版本是4.2，我们知道Array类型是值类型，其实1.0的时候Array 是引用类型，为什么要改成了值类型了呢？

{% highlight swift %}
// Swift 1.0
let arr = [5, 6]
arr[0] = 7 // error

// Swift 2.0 后
let arr = [5, 6]
arr[0] = 7 // error
{% endhighlight %}

上面同样的代码，当arr是引用类型的时候，数组的内容存放在堆上，此时arr其实是一个指针，指向堆上的一块连续内存存放着两个数字，arr[0] 其实类似于 *arr, 也就是现在我们做的操作是通过地址改变堆上存储的数据，这应该完全是没有任何问题的，但是在1.0中，因为 arr 被我们指定为 let 不可变，所以编译器就报错了。

> 例子中的arr是指针但是并没有直接指向堆中存储数据的地址，所以不需要写 `*`。

{% highlight cpp %}
int aa = 5;
int bb = 7;

int *p1 = &aa;
p1 = &bb;
*p1 = 10;

int * const p2 = &aa;
p2 = &bb; // error
*p2 = 10;

const int *p3 = &aa;
p3 = &bb;
*p3 = 10; // error
{% endhighlight %}

上面这个例子相信大家都知道，默认的p1既可以改变指向地址，也可以改变指向地址中的值。而p2则类似 arr，虽然不允许改变指向地址，但是应该允许改变指向地址中的值。如果需要不允许改变指向地址中的值，则就是p3。所以1.0的时候，Swift Array类型的设计就出错了。

到2.0后修正了这个问题，解决方案就是将Array由引用类型改成了值类型，此时arr并不是一个指针了，存储两个数字的地方并不是堆了，所以此时指定arr为 let 不可变后，自然也就不能改arr[0]了。

值类型和引用类型还有一个区别就是在赋值的时候:

{% highlight swift %}
struct A {
    var name: String = "xxx"
}

class B {
    var name: String = "xxx"
}

let a = A()
let b = B()

var aa = a
var bb = b

aa.name = "nancy"
bb.name = "nancy"


print("a name \(a.name), b name \(b.name).")
print("aa name \(aa.name), bb name \(bb.name).")

output:
a name xxx, b name nancy.
aa name nancy, bb name nancy.
{% endhighlight %}

我们可以发现结构体在赋值的时候，修改一个对另一个没有影响，是因为在赋值的时候，值类型会进行拷贝一份，也就是数据没有被共享；而类则会影响之前的，因为引用类型其实共享了同一份存储在堆上的数据。

> 虽然Swift中Array是值类型，但是Swift有做优化，并不是每次都会拷贝，当改变的时候才会拷贝，否则内部还是会共享同一份数据。

是不是看了之前的答案还是不知道如何选择，下面让我们看一下苹果给出的几点建议吧:

- 优先考虑struct

因为Swift 中的struct添加了很多功能，而且值类型更加安全，所以苹果推荐我们优先使用struct。

- 使用class当需要和OC交互时

- 使用class当需要保持同一性

- 使用struct当不需要保持同一性

- 优先考虑struct和protocol的组合来代替类的继承共享特性

当我们建立数据体系结构的时候，苹果推荐我们使用协议，因为协议既适用于struct也适用于class。

## 2019-09-14 补充

对于一个对象是选择struct还是class之前，首先是确定这个对象的语义。

所谓值语义对象的特征是内容和计算，值对象从语义是不可变的。

引用语义对象的特征是标识，状态，行为，引用对象从语义上是不可拷贝的。

```
这里从语义的不可变是不是可以理解为在进行赋值的时候，s2其实是s1的一份内容拷贝，所以当修改s2内部的成员不会影响到s1。
应该就是另一篇文章中所说的`copy by value`。

相对应的引用对象语义的不可拷贝，是不是可以理解为c1赋值给c2，其实他们指向的还是堆上同一块地址。

2021-04-09 更新：

不可拷贝应该是堆上的分配的对象，只要一份，不能进行拷贝。可以进行的操作只能改变指向这块内存的指针。

不可变是struct对象本身是不可变的，可变的是variable，也就是变量，当进行赋值的时候，其实并不是改了struct，而是生成了一个新的struct赋值给了variable。
```

当这个对象有状态的变化时，此时只能使用引用语义，对应的也就只能选择class。其他情况则可以使用struct。拿HTTP请求来说，Request因为有状态，初始化，发送中，发送成功等，所以必须用class；对应的Response，只是一个请求的返回内容而已，所以可以使用struct。

---

前面提到引用对象从语义上是不可拷贝的，可能会联想到NSObject的copy方法。

而NSObject的copy方法和mutableCopy方法则是对应两个协议：

{% highlight objc%}
@protocol NSCopying

- (id)copyWithZone:(nullable NSZone *)zone;

@end

@protocol NSMutableCopying

- (id)mutableCopyWithZone:(nullable NSZone *)zone;

@end
{% endhighlight %}

自定义子类可以实现这两个协议，因为调用copy和mutableCopy，就是去调用这两个协议的方法，NSObject没有实现这两个协议，所以对NSObject调用copy和mutableCoy会出现`unrecognized selector sent to instance`错误。

{% highlight objc%}
+ (id)copy {
    return (id)self;
}

+ (id)copyWithZone:(struct _NSZone *)zone {
    return (id)self;
}

- (id)copy {
    return [(id)self copyWithZone:nil];
}

+ (id)mutableCopy {
    return (id)self;
}

+ (id)mutableCopyWithZone:(struct _NSZone *)zone {
    return (id)self;
}

- (id)mutableCopy {
    return [(id)self mutableCopyWithZone:nil];
}
{% endhighlight %}

所谓深拷贝和浅拷贝，给出一张Apple的示意图：

![swift_struct_and_class_1]({{site.url}}/assets/images/blog/swift_struct_and_class_1.png)

主要区别在于处理引用对象。

最后给出NSString和NSMutableString的copy协议的实现。

{% highlight objc%}
// NSString
- (id)copyWithZone:(NSZone *)zone {
    if (NSStringClass == Nil)
        NSStringClass = [NSString class];
    return RETAIN(self);
}
- (id)mutableCopyWithZone:(NSZone*)zone {
    return [[NSMutableString allocWithZone:zone] initWithString:self];
}

// NSMutableString
-(id)copy {
    return [[NSString alloc] initWithString:self];
}
-(id)copyWithZone:(NSZone*)zone {
    return [[NSString allocWithZone:zone] initWithString:self];
}
{% endhighlight %}

NSString的copy就是简单的retain，当然自定义子类的时候，`copyWithZone:`方法完全可以新建一个对象。

## References

[https://docs.swift.org/swift-book/LanguageGuide/ClassesAndStructures.html](https://docs.swift.org/swift-book/LanguageGuide/ClassesAndStructures.html)

[https://www.mikeash.com/pyblog/friday-qa-2015-07-17-when-to-use-swift-structs-and-classes.html](https://www.mikeash.com/pyblog/friday-qa-2015-07-17-when-to-use-swift-structs-and-classes.html)

[https://www.cnblogs.com/weidagang2046/archive/2010/07/24/value-vs-ref.html](https://www.cnblogs.com/weidagang2046/archive/2010/07/24/value-vs-ref.html)

[https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/ObjectCopying.html](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/ObjectCopying.html)
