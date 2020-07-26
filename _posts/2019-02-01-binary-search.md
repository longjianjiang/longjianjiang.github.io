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

二分法在查找中使用非常多，不过使用二分有一个前提就是列表有序，关于排序可以参考笔者之前的[笔记](http://www.longjianjiang.com/sort/)。

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

第一种思路和当查找中存在重复元素思路是一致的，就是找**第一个大于等于target**的元素，尝试往左对mid进行二分查找，找到第一个大于等于target的元素，参考代码如下:

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

第二种思路相对简洁，没有上一步的判断尝试对mid进行二分，假定right指向第一个大于等于target的元素，当然也有可能不存在，也就是列表中所有元素都小于target。其实判断条件还是和第一种思路是一致的，分两种情况:

- lists[mid] < target
此时我们需要继续在列表右半段查找，即 left = mid+1。

- lists[mid] >= target
此时我们需要继续在列表左半段查找，即 right = mid。

后面每一次二分，如果走到 `lists[mid] >= target` 分支，其实就等于往左对mid进行二分，而right始终指向的数字都是大于等于target的，所以最后我们返回right即可，当right等于列表长度时，证明列表中的元素都小于target。

参考代码如下:

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

第一种尝试对mid往右继续进行二分，参考代码如下:

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

第二种则假定right指向第一个大于target的元素，此时right的前一个元素就是最后一个小于等于target的元素，二分的判断条件依然和第一种方式一致：

- lists[mid] <= target

mid指向的元素不大于target，而我们希望right指向大于target的元素，所以此时我们需要在列表的右半段继续寻找，即 left = mid+1。

- lists[mid] > target

mid指向的元素大于target了，而我们希望right指向的是第一个大于target的元素，所以此时我们需要在列表中左半段继续查找，即 right = mid。

后面的每一次二分，如果走到 `lists[mid] <= target`，其实就等于往右对mid进行二分，尝试寻找比target大的元素，
如果走到 `lists[mid] > target`，则是往左寻找第一个大于target的元素，此时我们不难发现right指向的元素是大于target的，所以最后right指向的就是第一个大于target的元素，而right-1指向的就是最后一个小于等于target的元素，所以返回right-1即可。

参考代码如下:

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

其实我们可以发现，这里查找**第一个大于target的元素**和之前查找**第一个大于等于target的元素**在代码上只改变了一个地方，就是将之前的 `lists[mid] < target` 改成了 `<=`。

- 当列表中没有重复元素的时候，效果时一致的

比如[3, 5, 6, 8, 9, 10]，target为7，符合要求的元素是8，idx为3。

- 当列表中有重复元素的时候，效果就不一样了

比如[0, 1, 1, 1, 1, 1]，target为1。造成的区别其实是条件的不同导致查找方向的不同：

当 `<` 时，也就是查找第一个大于等于target元素时，找到mid后，大于等于1，尝试往左继续寻找；
当 `<=` 时，也就是查找第一个大于target的元素，找到mid后，小于等于1，并不大于1，所以尝试往右继续寻找；

因为寻找方向的不同，导致了最后结果的不同。


# STL

STL 中也提供了三个常用的查找方法。

`binary_search()` 给定左闭右开的区间中查找target是否存在;

`upper_bound()` 给定左闭右开的区间中查找第一个大于target的迭代器;

`lower_bound()` 给定左闭右开的区间中查找第一个不小于target的迭代器，也就是第一个大于等于target的；

upper_bound 和 lower_bound 所指定的区间需要是有序的，内部实现就是使用二分进行查找的；

# 三分

二分的解决问题其实类似一个一次函数，单调递增，所以这也是为什么二分的前提是有序。

假设现在问题类似是一个二次函数，求最值，此时就需要用到三分。

mid = (Left + Right) / 2
midmid = (mid + Right) / 2;
如果mid靠近极值点，则Right = midmid；
否则(即midmid靠近极值点)，则Left = mid;
