---
layout: post
title: "单调栈"
date: 2021-02-26
excerpt: "所谓单调栈，首先是一个栈，其次就是栈内元素是升序或者降序"
tag:
- Algorithm
comments: true
---

# 单调栈

所谓单调栈，首先是一个栈，其次就是栈内元素是升序或者降序。

当某些问题需要保持有序序列的时候，依次**入栈**，当新的元素打破了当前有序，此时需要**出栈**。出栈一般都是返回问题的结果。

## 寻找第一个比自己大的数
看一个实际的例子，给定数组[6 4 3 2 5]，要求返回数组中每个元素下一个更大元素，没有返回-1。

最直接的方法就是两次循环，找到第一个比当前元素最大的元素。但是如何做到只遍历一次呢？一般做到时间复杂度的降低，需要使用额外的数据结构。

想一下，题目要求比当前元素大的第一个元素，那么需要是升序序列，那么其实，直接依次返回下一个元素就好。如果是倒序，那就得看后面是否存在打破倒序的元素构成局部倒序，此时就可以找到更大的元素。

所以我们创建一个栈来存储倒序的序列，当出现新的元素大于栈顶元素时，此时出栈同时记录栈顶元素的下一个大元素就是要进栈的元素。这里需要注意的一点时，因为栈中存储的事倒序的序列，所以出栈的时候是一个while循环，需要把栈中所有符合条件的元素都出栈。

拿例子来说：

```
1. 首先栈是空的，入栈，[6];
2. 入栈元素4，小于栈顶6，入栈，[6, 4];
3. 入栈元素3，2，同理，入栈，[6, 4, 3, 2];
4. 入栈元素5，此时5大于当前栈顶2，所以2出栈，[6, 4, 3]; 
	此时因为有while循环继续将栈顶小于5的元素出栈，最后将5入栈，[6, 5];
```

最后会发现栈中还留了两个元素，为了让这两个元素也可以出栈，可以往元数组中插入一个辅助数字INT_MAX，这样就可以按之前的逻辑进行处理。

## 最大矩形问题

继续另一个问题，给定 n 个非负整数，用来表示柱状图中各个柱子的高度。每个柱子彼此相邻，且宽度为 1 。求在该柱状图中，能够勾勒出来的矩形的最大面积。
给定数组[2,1,5,6,2,3]，最大面积是10。

最直接的方案依然是两次for循环找出所有可能的结果，返回最大的。

求矩形面积问题，要确定高和宽。依然是升序和降序的情况，当升序的时候，面积是可能会更大，所以入栈，当出现降序，此时需要计算栈内当前元素构成的面积同时出栈。

下面就是出栈的时候，如何确定矩形的高和宽，高其实就是出栈元素的值。这里宽的计算用到了当前插入元素的位置，拿例子来说：

```
1. 首先栈是空的，入栈，[2];
2. 入栈元素1，小于栈顶2，出栈元素2，栈为空。此时矩形的高度为2，宽度呢？我们先当1来处理。
	area = 2 * 1；
3. 入栈元素5，大于栈顶1，入栈，[1, 5];
4. 同理入栈元素6，[1, 5, 6];
5. 入栈元素2，小于栈顶6，出栈元素6，此时矩形高度为6，宽度呢？
	如何确定宽度，确定了左右边界，右边界-左边界就是宽度，而右边界其实就是入栈元素的idx，左边界是栈顶元素+1，所以宽度就是i-(stk.top()+1)。
	area = 6 * (4 - (2+1);
	栈顶元素5大于2，继续出栈，高度是5，宽度是(4 - (1+1));
	area = 5 * 2;
	此时栈[1, 2];
6. 入栈元素3，大于栈顶2，入栈，[1, 2, 3];
7. 最后入栈0，来计算栈中所有元素；
	首先是3出栈，高度是3，宽度是(6 - (4+1))，area = 3 * 1；
	2出栈，高度是2，宽度是(6 - (1+1))，area = 2 * 4；
	1出栈，高度是1，宽度是多少呢？此时栈为空，宽度还是1嘛？
	明显不是1，因为元素1时最低的元素，其他元素都比1大，所以宽度其实应该是当前入栈的索引，也就是6。
```

## 小结

几个注意的点：

- 出栈需要循环判断，将栈中符合条件的所有元素出栈。

- 插入辅助元素进行处理。

- 使用插入元素的位置来计算宽度。

我认为使用这个结构有两点需要注意的，一是确定使用升序还是降序去解决问题；二是局部不满足有序的时候，此时出栈需要做什么事情。

不过最重要的还是，理解题目，分析题目背后的问题是如何可以利用这样一个有序的数据结构来降低时间复杂度。

# References

[ref](https://www.cnblogs.com/boring09/p/4231906.html)