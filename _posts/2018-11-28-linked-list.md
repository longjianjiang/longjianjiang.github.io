---
layout: post
title:  "链表"
date:   2018-11-28
excerpt:  "本文是笔者总结链表的笔记"
tag:
- Algorithm
comments: true
---

> 最近笔者在刷算法题，Leetcode上链表分类的已经刷完，本文算是一个小小的总结。

## 数据结构

链表是一种离散型的存储数据结构，通过指针将节点连接起来。 一个单链表数据结构如下：

```
class ListNode
    attr_accessor :val, :next
    def initialize(val, nextNode=nil)
        @val = val
        @next = nextNode
    end
end
```





