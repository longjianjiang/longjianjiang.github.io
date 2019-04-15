---
layout: post
title:  "内存对齐"
date:   2016-08-26
excerpt: "所谓内存对齐，其实是为了加快CPU读取数据的速度，因为CPU读取数据是按块(X64架构的计算机是8个字节为一个块)来读取的，所以按照一定的对齐规则存储数据，会大大提高CPU的读取效率。"
tag:
- Algorithm
comments: true
---

## 先看一个例子！

![Snip20160826_1.png]({{site.url}}/assets/images/blog/memoryAline_1.png)

>我们发现两个结构体中存放的是相同类型的三个数据，只是顺序有所不同，通过`sizeof()`函数计算出所占字节数目也完全不同。
其实我们在学习C语言的时候应该已经说过这种现象是因为内存对齐所导致的，不过下面我们来进一步了解下内存对齐的规则。

**PS:阿里校招的测评有一题就是考了这个，见下图：**

![Snip20160826_2.png]({{site.url}}/assets/images/blog/memoryAline_2.png)

## 为什么需要内存对齐
所谓内存对齐，其实是为了加快CPU读取数据的速度，因为CPU读取数据是按块(X64架构的计算机是8个字节为一个块)来读取的，所以按照一定的对齐规则存储数据，会大大提高CPU的读取效率。

更多内容关于内存对齐对性能请参考[文章](http://www.ibm.com/developerworks/library/pa-dalign/ )

> 举个例子
在32位系统中，假如一个int变量在内存中的地址是0x00ff42c3,因为int是占用4个字节，所以它的尾地址应该是0x00ff42c6，这个时候CPU为了读取这个int变量的值，就需要先后读取两个块，分别是0x00ff42c0~0x00ff42c3和0x00ff42c4~0x00ff42c7，然后通过移位等一系列的操作来得到。但是如果编译器对变量地址进行了对齐，比如放在0x00ff42c0，CPU就只需要一次就可以读取到，这样的话就加快读取效率。

还有就是不同平台下的硬件原因，一些硬件平台只能在某些地址处取某些特定类型的数据，否则抛出硬件异常，并不是所有的硬件平台都能访问任意地址上的任意数据的。

## 内存对齐的规则
- 基本数据类型对齐
基本数据类型是按照其类型所占字节数来对齐的，需要注意的是，不同平台下，有些数据类型所占字节数是不一样的，比如
>指针类型在64位机器上占8个字节，在32位机器上则占4个字节。

- 结构数据类型对齐
结构体内的各个数据对齐。在结构体中的第一个成员的首地址等于整个结构体的变量的首地址，而后的成员的地址随着它声明的顺序和实际占用的字节数递增。为了总的结构体大小对齐，会在结构体中插入一些没有实际意思的字符来填充（padding）结构体。下面给出规则：
> 1、对于结构的各个成员，第一个成员位于偏移为0的位置，以后每个数据成员的偏移量必须是`min(#pragma pack())指定的数，这个数据成员的自身长度)` 的倍数。
~~ 2、在数据成员完成各自对齐之后，结构(或联合)本身也要进行对齐，对齐将按照`min(#pragma pack指定的数值,结构(或联合)最大数据成员长度)`进行对齐。~~

**#pragma pack()的预处理指令可以修改系统默认的对齐数，取值为1，2，4，8，16**

## 总结
**PS:联合的大小是联合中所占字节最多的成员**
了解完规则之后，前面的例子加阿里的题目也就迎刃而解了。


# alignas & alignof 更新于2019-04-15

C++11中提供了 `alignas` 来设置对齐。一个常用的例子如下:

{% highlight cpp %}
alignas(double) unsigned char buf[sizeof(double)];
{% endhighlight %}

上面我们声明了一个buf数组，大小是`sizeof(double)`，通过`alignas(double)`设置了和double一样的对齐大小，此时buf数组可以存储一个double型数据。

之前讲到`#pragma pack()`使用这个预处理也能修改对齐的大小，那么和使用alignas有什么区别呢？

1.`#pragma pack()`设置的全局的对齐方式，而alignas则是只作用于某个变量或者结构；

2.`#pragma pack()`有最大上限，为类型的默认对齐大小，而alignas有最小下限，也就是最少需要设置为默认类型的对齐大小，同时需要是2的幂。

下面来看几个例子：

> 默认结构体的对齐大小为4字节，实际根据成员的不同，最终的对齐大小和占最多字节成员的对齐大小一致。

{% highlight cpp %}
struct alignas(8) MyStruct
{
    /*alignas(8)*/ char member;
    /*alignas(8)*/ int a;

}mystruct;

void main() {
    cout << sizeof(mystruct) << ", " << alignof(mystruct) << endl; // 8, 8
    cout << offsetof(MyStruct, member) << ", " << offsetof(MyStruct, a) << endl; // 0, 4
}
{% endhighlight %}

用alignas指定了结构的对齐，所以并不是4而是8；

{% highlight cpp %}
struct MyStruct
{
    alignas(8) char member;
    alignas(8) int a;

}mystruct;

void main() {
    cout << sizeof(mystruct) << ", " << alignof(mystruct) << endl; // 16, 8
    cout << offsetof(MyStruct, member) << ", " << offsetof(MyStruct, a) << endl; // 0, 8
}
{% endhighlight %}

用alignas指定结构成员的对齐，所以结构的大小变成了16；

{% highlight cpp %}
struct alignas(16) MyStruct
{
    /*alignas(8)*/ char member;
    /*alignas(8)*/ int a;

}mystruct;

void main() {
    cout << sizeof(mystruct) << ", " << alignof(mystruct) << endl; // 16, 16
    cout << offsetof(MyStruct, member) << ", " << offsetof(MyStruct, a) << endl; // 0, 4
}
{% endhighlight %}

用alignas修改结构的对齐，此时也影响了结构的大小。

> 结构的大小是结构对齐大小的整数倍。

{% highlight cpp %}
alignas(8) unsigned char buf[2000];
cout << sizeof(buf) << ", " << alignof(buf) << endl; // 2000, 8
{% endhighlight %}

虽然buf设置对齐是8，说明数组的地址能被8整除，但是这并不会改变数组的大小。

## References

[https://www.cnblogs.com/ye-ming/p/9295460.html](https://www.cnblogs.com/ye-ming/p/9295460.html)