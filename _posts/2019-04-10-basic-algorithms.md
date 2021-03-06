---
layout: post
title:  "基本算法"
date:   2019-04-10
excerpt:  "本文是笔者学习基本算法的笔记"
tag:
- Algorithm
comments: true
---

> 笔者最近在刷Leetcode算法题，在这里笔者将自己对常见算法的学习和总结记录下来。

## 枚举

枚举应该是最直观的，通常循环遍历即可。

## 深搜和回溯

笔者在刷Leetcode中 `Combination Sum` 这类题以及求数独时，接触到深搜和回溯是用来解决这类问题的。

这类问题都有一个特点，都是要求获得满足要求的一组解。所以此时求解的过程就类似走迷宫，从起始点选择一条路线出发，当遇到路口时选择一个继续往前走(深搜)，直到发现此条路不通，此时原路返回到上一个路口(回溯)，在改路口选择另一个路线继续尝试，反复此过程，直到走完所有可能的路线。

解决上述问题的关键就在于定义`dfs`深搜方法，一个伪代码如下:

{% highlight ruby %}
def dfs(arr, idx, ...)
	return if some_condition

	for i in idx...arr.size
		obj = arr[i]

		choose()

		dfs(arr,i+1, ...)

		unchoose()
	end
end
{% endhighlight %}

## 动态规划

动态规划的题很多，一般如果可以将原问题分解为若干个规模较小的子问题，解决了这些子问题，也就是解决了原问题，此时就可以使用动态规划。

所以动态规划的难点在于去寻找某个问题是否可以分解为子问题，进而获得一个所谓的递推公式，这个公式就描述了如何通过子问题去求原问题。

动态规划的优点在于可以缓存子问题的解，减少了不必要的计算，同时也可以进行求解最优化。

如果动态规划中每次只用上一步的结果，我们可以使用覆盖的方式，这样可以减少一维的空间复杂度。

## 贪心

贪心算法同样需要将一个问题可以分解成子问题，在解决每个子问题时每次都采取对本次最有利的选择。

举个例子，找零钱，为了使币的数量最少，每次都尝试从大的面值开始寻找，如果太大了，继续找次之的。

但是上述找零钱例子中，按照这种策略有时候不一定是最优，比如当可供选择的面值为`10, 8, 5, 1`，此时需要找16块的零钱，按贪心的做法`16=10+5+1`需要3个币，而实际上`16=8+8`只需要2个币即可。

上述问题其实是一个局部最优和全局最优的问题，贪心算法适用于最优子结构问题中，最优子结构就是局部最优可构成全局最优。

## 分治

归并排序是分治法的一个典型应用，将问题每次拆分为两个规模较小的问题，直到不能拆分此时也就是解决了子问题；这个时候将所有解决的子问题合并最后也就解决了原规模较大的问题。

这里需要注意的是拆分出来的子问题彼此没有耦合，也就是说求解子问题不依赖其他子问题的解，否则就不适用于归并算法。

## 递归

所谓递归就是方法内部调用自身，一个常见的例子就是斐波那契数列的实现；

尾递归则是比递归多了一个参数，用来存放上一次函数调用的结果，这样其实每一次的递归调用都在收集结果，这样也就有了尾递归优化，不用每次都压栈，使用一层栈就可以了。

[ref](https://farer.org/2017/03/10/tail-recursion/)

## 总结

基本的算法是每一个开发者的必备技能，为了更好的理解这些算法，还得不断的做题检验自己。
