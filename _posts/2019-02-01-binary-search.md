---
layout: post
title:  "二分查找"
date:   2019-02-01
excerpt:  "本文是笔者总结二分查找的笔记"
tag:
- Algorithm
comments: true
---

> 笔者最近在刷Leetcode算法题，很多题目都和二分查找有关，所以在这里笔者将自己二分查找的学习和总结记录下来。

二分发在查找中使用非常多，不过使用二分有一个前提就是列表有序，关于排序可以参考笔者之前的[笔记](http://www.longjianjiang.com/sort/)。

{% highlight ruby %}
# @param {Integer[]} lists
# @param {Integer} target
# @return {Integer}
def bsearch(lists, target)
    left, right = 0, lists.length-1
    while left <= right
        mid = left + (right-left) / 2
        if lists[mid] == target
            return mid
        elsif lists[mid] < target
            left = mid+1
        else
            right = mid-1
        end
    end

    -1
end
{% endhighlight %}

上面是笔者给出的一个通用二分查找，查找数组中某个特定的值。

但是实际的查找会有很多附加条件，而且数据多种多样，所以上面的查找在一些特定条件下就不准确了，比如下面几种情况：

## 当数组中有重复元素，查找最先出现的元素或者最后出现的元素

对于这种情况其实还是比较简单的，尝试对mid继续进行二分查找即可，参考代码如下

{% highlight ruby %}
# @param {Integer[]} lists
# @param {Integer} target
# @return {Integer}
def bsearch_1(lists, target)
    left, right = 0, lists.length-1
    while left <= right
        mid = left + (right-left) / 2
        if lists[mid] == target
            if mid == 0 || lists[mid-1] != target
                return mid
            else 
                right = mid-1
            end
        elsif lists[mid] < target
            left = mid+1
        else
            right = mid-1
        end
    end

    -1
end
{% endhighlight %}

## 查找第一个大于等于target的元素

> 此时target不一定出现在列表中，所以就没有必要写 `target == list[mid]` 的判断了;

这里有两种思路:

第一种思路就是找**第一个大于等于target**的元素，尝试对mid进行二分查找，找到第一个大于等于target的元素，参考代码如下:

{% highlight ruby %}
# @param {Integer[]} lists
# @param {Integer} target
# @return {Integer}
def bsearch_2(lists, target)
    left, right = 0, lists.length-1
    while left <= right
        mid = left + (right-left) / 2
        if lists[mid] >= target
            if mid == 0 || lists[mid-1] < target
                return mid
            else
                right = mid-1
            end
        else
            left = mid+1
        end
    end

    -1
end
{% endhighlight %}

第二种思路就是找**最后一个小于target**的元素，这样这个元素的后面就是第一个大于等于target的元素了，所以我们就让right指向这个位置，参考代码如下:

{% highlight ruby %}
# @param {Integer[]} lists
# @param {Integer} target
# @return {Integer}
def bsearch_3(lists, target)
    left, right = 0, lists.length
    while left < right
        mid = left + (right-left) / 2
        if lists[mid] < target
            left = mid+1
        else
            right = mid
        end
    end

    right
end
{% endhighlight %}

## 查找最后一个小于等于target的元素

和上一种情况类似，同样的有两种实现方式。

第一种尝试对mid继续进行二分，参考代码如下:

{% highlight ruby %}
# @param {Integer[]} lists
# @param {Integer} target
# @return {Integer}
def bsearch_4(lists, target)
    left, right = 0, lists.length-1
    while left <= right
        mid = left + (right-left) / 2
        if lists[mid] <= target
            if mid == lists.length-1 || lists[mid+1] > target
                return mid
            else
                left = mid+1
            end
        else
            right = mid-1
        end
    end

    -1
end
{% endhighlight %}

第二种则首先找到第一个大于target的元素，那么这个元素的前一个就是最后一个小于等于target的元素，让right指向第一个大于target的元素，那么right-1指向的元素就是最后一个小于等于target的元素，参考代码如下:

{% highlight ruby %}
# @param {Integer[]} lists
# @param {Integer} target
# @return {Integer}
def bsearch_3(lists, target)
    left, right = 0, lists.length
    while left < right
        mid = left + (right-left) / 2
        if lists[mid] <= target
            left = mid+1
        else
            right = mid
        end
    end

    right-1
end
{% endhighlight %}

