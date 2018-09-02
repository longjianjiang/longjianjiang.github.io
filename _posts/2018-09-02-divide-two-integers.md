---
layout: post
title:  "29. Divide Two Integers"
date:   2018-09-02
excerpt:  "本文是笔者的对题目的一个学习思路"
tag:
- Algorithm
comments: true
---

本题要求实现int的除法，不能使用 `/` 操作符。

## 解题思路

想到可以使用位运算符来实现，因为除法其实可以转化为减法，而减法都是通过加法来实现的。

### 加法

我们知道数字在计算机中以二进制的形式存储，所以做加法也就是通过二进制的位运算符来实现的。

所以加法其实分为两部分，一部分时和，一部分为进位，而求和我们可以使用 `^` 也就是异或运算符， 进位则可以使用 `&` 也就是与运算符。

所以用上述两个位运算符即可实现加法，代码如下：

```
int add(int a, int b) {
  if (b == 0) return a;
  sum = a ^ b;
  carry = (a & b) << 1;
  return add(sum, carry);
}
```

### 减法

因为计算机中数字是以补码的形式存储的，所以做减法，其实就是加上这个数的补码，这样减法也转化为加法的运算。

正数的补码的就是正数本身，负数的补码是对正数求反码然后加1。

所以减法实现如下：

```
int subtract(int a, int b) {
  return add(a, add(~b, 1));
}
```

### 除法

其实我们做除法，就是判断除数可以减多少次被除数，当除数小于被除数，此时除法的商和余数都可以知道了。

所以除法实现如下：

```
int divide(int a, int b) {
  // 先取两数的绝对值，计算出结果，最后再判断奇偶性

  int x = a > 0 ? a : add(~a, 1);
  int y = b > 0 ? b : add(~b, 1);

  int left = x;
  int quotient = 0;

  while (left >= y) {
      left = subtract(left, y);
      quotient = add(quotient, 1);
  }

  if ( (a^b) < 0) {
      quotient = add(~quotient, 1);
  }

  return quotient;
}
```

不过上述实现除法的方法，存在效率问题，如果除数很大，被除数很小，会执行很多次。

所以我们需要减少减法的次数，其实我们可以先找被除数最大的倍数（小于除数），做减法，然后依次减小倍数，直到除数小于被除数。

改进后的除法实现如下：

```
int divide(int a, int b) {
  // 先取两数的绝对值，计算出结果，最后再判断奇偶性

  long long x = a > 0 ? a : add(~a, 1);
  long long y = b > 0 ? b : add(~b, 1);

  long long tmp = 0;
  long long result = 0;

  for(int i = 31; i >= 0; --i) {
      tmp = y << i;
      if (x >= tmp) {
          result = add(result, 1<<i);
          tmp = subtract(x, tmp);
      }
  }

  if ( (a^b) < 0) {
      result = add(~result, 1);
  }

  return result;
}
```

