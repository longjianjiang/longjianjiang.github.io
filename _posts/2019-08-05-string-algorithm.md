---
layout: post
title: "字符串算法"
date: 2019-08-05
excerpt: "nope"
tag:
- Algorithm
comments: true
---

> 笔者在这里记录下字符串相关的算法。

## KMP

给定字符串s和子串p，让我们寻找p在s中的位置。

使用传统的暴力方法，我们使用两个指针来进行遍历，假设i指向s，j指向p。当s[i] != p[j] 的时候，此时i和j都要回退进行下一次的寻找。所以当s很长的时候，就很浪费时间，因为进行来很多重复的比较。下面给出一个参考代码：

{% highlight cpp %}
{% endhighlight %}


