---
layout: post
title:  "4. Median of Two Sorted Arrays"
date:   2018-04-18
excerpt:  "本文是笔者的对题目的一个学习思路"
tag:
- Algorithm
comments: true
---

## 解题思路

> 求两个数组的中间值，分奇偶情况。


### 方法一

最直觉的思路就是先将两个数组合并后排序，然后进行求中间值，但是排序的最小复杂度是O(n * log(n)),所以并不满足题目中要求的时间复杂度 O(log (m+n))。



### 方法二

笔者开始并没有想到该方法，查阅资料后了解，所以在此记录下解题思路：

使用二分的思路在两个数组中寻找第k个值，假设k/2下标两个数组都存在，那么将数组中下标为k/2的值进行比较

1、如果nums1[k/2-1] > nums2[k/2-1], 那么nums2的前k/2个就可以忽略了，因为想象如果数组是合并后的，要寻找第k个，那么k/2位置的肯定小于第k个，那么说明第k个肯定在两个数组的后面元素中，所以可以忽略nums2的前k/2个元素；

2、如果nums1[k/2-1] < nums2[k/2-1], 同理nums1中的前k/2个就可以忽略了;

同时有两种特殊的情况可以直接取值:

1、当其中一个数组为空，那么第k个元素就是nums[k-1];

2、当k=1, 也就是求第一个元素的情况，此时肯定是取数组中首元素较小的那个，因为同样的想象如果数组是合并后的，要找第一个元素，那么在有序数组中肯定是取首元素;

所以综上可以使用递归的思路进行求两个数组中第k个元素。


下面是笔者使用ruby解答的答案:


```
def find_kth(nums1, nums2, k)
  m = nums1.length
  n = nums2.length

  return find_kth(nums2, nums1, k) if m > n # assume nums1 length less than nums2 length
  return nums2[k-1] if m == 0
  return [nums1[0], nums2[0]].min if k == 1

  i = [m, k/2].min
  j = [n, k/2].min

  if nums1[i-1] > nums2[j-1]
    return find_kth(nums1, nums2[j...n], k-j)
  else
    return find_kth(nums1[i...m], nums2, k-i)
  end
end


def find_median_sorted_arrays(nums1, nums2)
  m = (nums1.length + nums2.length + 1) / 2
  n = (nums1.length + nums2.length + 2) / 2

  return ( find_kth(nums1, nums2, m) + find_kth(nums1, nums2, n) ) / 2.0
end
```
