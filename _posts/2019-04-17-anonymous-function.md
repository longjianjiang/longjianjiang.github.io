---
layout: post
title:  "匿名函数笔记"
date:   2019-04-17
excerpt:  "本文是笔者学习匿名函数的笔记"
tag:
- iOS
comments: true
---

Block是C的一个扩展，所以在C++中同样可以使用，同时C++11新增了lambda表达式，这两者其实都是匿名函数的实现。下面首先来看block。

## block

### 类型

> block其实就是一个对象，因为它有`isa`。

{% highlight cpp %}
int main() {
    auto block = ^int(int a, int b) {
        return a + b;
    };
    cout << block(5, 7) << endl;
    return 0;
}
{% endhighlight %}

上述就是一个简单的block，可以看到使用起来和函数很像，同时我们都知道block也可以作为函数参数，函数返回值。所以block是怎样一个类型呢？

block有三种类型，对应着block对象存储的不同内存位置。

```
_NSConcreteStackBlock   
_NSConcreteMallocBlock	
_NSConcreteGloalBlock
```

那么具体block里面有那些东西呢，他是怎么执行创建时候写的代码块呢？

{% highlight cpp %}
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static int __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {
    return a + b;
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main() {
    auto block = ((int (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    ((int (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 5, 7);
    return 0;
}
{% endhighlight %}

上面代码经过`clang -rewrite-objc xxx.m`变成了上述一大段代码。

可以看到block对象是一个结构体，结构体有:

1.有isa指针指向上述所说的三种类；

2.有一个FuncPtr指针指向了函数，这个函数就是block代码块生成的一个函数；

3.有一个desc结构体来描述block的信息;

目前基本对block有了一个新的认识，继续往下看。

### 捕获变量

### 内存管理

### 小结

## lambda


## 总结

## References

[iOS与OS X多线程和内存管理](https://book.douban.com/subject/24720270/)
[https://www.mikeash.com/pyblog/friday-qa-2011-06-03-objective-c-blocks-vs-c0x-lambdas-fight.html](https://www.mikeash.com/pyblog/friday-qa-2011-06-03-objective-c-blocks-vs-c0x-lambdas-fight.html)