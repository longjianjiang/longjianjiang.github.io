---
layout: post
title:  "5. Longest Palindromic Substring"
date:   2018-04-30
excerpt:  "本文是笔者的对题目的一个学习思路"
tag:
- Algorithm
comments: true
---


## 解题思路

> 我所能想到最直觉的思路就是两次遍历，以当前为中心，向两边进行寻找，每次记录起点和长度，同时注意奇偶情况

### 方法1

下面给出笔者使用ruby实现的答案：


```
def search(s, left, right, idx, len, vars)
  step = 1
  while left - step >= 0 && right + step < s.length
    break if s[left - step] != s[right + step]
    step += 1
  end

  wide = right - left + (2 * step - 1)
  if wide > len
    eval "len = #{wide}", vars
    eval "idx = left - #{step} + 1", vars
  end
end


def longest_palindrome_(s)
  left = 0
  right = 0
  idx = 0
  len = 0

  for i in 0...s.length
    if s[i] == s[i+1]
      left = i
      right = i + 1
      search(s, left, right, idx, len, binding)
    end

    left = right = i
    search(s, left, right, idx, len, binding)
  end

  return s[idx, len]
end
```

### 方法二

笔者开始并没有想到该方法，查阅资料后了解，所以在此记录下思路：

> 该方法被称作`Manacher’s Algorithm`, 它将时间复杂度提升到了线性的O(n);

首先将在原字符串中依次插入一个特殊符号进行标记，比如 'abab' -> '#a#b#a#a#', 这样不论是奇数还是偶数都可以转化为奇数个；

```
s   = # a # b # a # b #

idx = 0 1 2 3 4 5 6 7 8

            c     r

p[] = 0 1 0 3
```

如上所示p数组中存放的是以下标为i为中心向两边扩张的最长距离，此时中心点为3，最右边界为6，我们继续往前计算p[4], 很明显p[4]==0, 我们发现p[2]同样的也是等于0，而且我们知道p[4]和p[2]以p[3]为中心对称，所以是不是p[5]==p[1]==1呢？

> 对称点 p[5] == p[3- (5-3)] == p[1]

```
s   = # a # b # a # b #

idx = 0 1 2 3 4 5 6 7 8

            c     r

p[] = 0 1 0 3 0 3
```

其实我们看下p[5]==3，并不等于其对称点p[1]==1. 因为我们还需要以5为中心向两边扩张计算出idx为5的最长距离(因为s[7]==s[3], s[8]==s[2])，从而进行更新中点；之前中点为3时，最右边界为6，现在idx为5时，最右边界为8，所以我们需要同时更新中心点和最右边界;


```
s   = # a # b # a # b #

idx = 0 1 2 3 4 5 6 7 8

                c     r

p[] = 0 1 0 3 0 3 0
```

此时和之前一样继续往前计算p[6], 先判断idx是否超过最右边界，如果没有就取边界与idx的差或对称点中较小的(因为快到字符串尾部时以对称点为基础就不准确，就像p[7]==1,而不是等于p[3]==3)，否则就是0； 然后以idx为中点往两边扩张进行寻找最长距离，从而判断是否需要更新中心点和最右边界；

笔者自己理解的大致思路就是这样，下面给出使用ruby实现的答案：

```
def addPrefix(s)
  result = '^'
  for i in 0...s.length
    result += "\##{s[i]}"
  end
  result += "\#\$"
end


def longest_palindrome(s)
  return '' if s.length == 0 || s == nil

  t = addPrefix(s)
  p = Array.new(t.length, 0)
  c = 0
  r = 0
  res_len = 0
  res_idx = 0

  for i in 1...t.length
    i_mirror = c - (i - c)
    p[i] = r > i ? [r-i, p[i_mirror]].min : 0

    while t[i + 1 + p[i]] == t[i - 1 - p[i]]
      p[i] += 1
    end

    if i + p[i] > r
      c = i
      r = i + p[i]
    end

    if p[i] > res_len
      res_len = p[i]
      res_idx = i
    end
  end

  puts p.to_s

  return s[((res_idx-1-res_len) / 2), res_len]
end
```
