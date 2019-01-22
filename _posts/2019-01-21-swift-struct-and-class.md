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

上面答案提到了值和引用，两者的本质区别其实是存储内容的地方不同。值类型的数据存放在栈上，而引用类型的数据则存放在堆上。

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

上面这个例子相信大家都知道，默认的p1既可以改变指向地址，也可以改变指向地址中的值。而p2则类似 arr，虽然不允许改变执行地址，但是应该允许改变执行地址中的值。如果需要不允许改变指向地址中的值，则就是p3。所以1.0的时候，Swift Array类型的设计就出错了。

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

## References

[https://docs.swift.org/swift-book/LanguageGuide/ClassesAndStructures.html](https://docs.swift.org/swift-book/LanguageGuide/ClassesAndStructures.html)

[https://www.mikeash.com/pyblog/friday-qa-2015-07-17-when-to-use-swift-structs-and-classes.html](https://www.mikeash.com/pyblog/friday-qa-2015-07-17-when-to-use-swift-structs-and-classes.html)