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

所谓捕获，就是匿名函数中使用了外部定义的变量。所以根据变量的定义位置以及类型有多种可能，下面一一介绍：

### 静态区变量

静态区变量包括静态局部变量，静态变量，全局变量。

{% highlight cpp %}
int num;
void main() {
    ^{ num= 7; };
}
{% endhighlight %}

block中使用静态区变量就很简单，不会往block结构体中增加成员，同样也可以修改。

### 局部变量

- 非对象类型

{% highlight cpp %}
struct __main_block_impl_0 {
  ...
  int num;
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int num = __cself->num; // bound by copy
  num; 
}

void main() {
    int num;
    ^{ num; };
}
{% endhighlight %}

当block中使用了一个局部变量时，此时生成的block结构体中就会加入这个局部变量作为一个新的成员，并且在block初始化的时候将外部的局部变量作为参数传递。

此时如果在block内部对局部变量进行修改是不允许的，其实道理很简单，block结构体里仅仅存了一个整形的数据，就算内部改了并不会更新外部，所以如果进行修改则直接报错。

不知道大家之前有没有注意到block根据代码块生成的一个函数，默认有一个参数`__cself`，一个指向block结构体的指针。它的作用就是为了在生成的函数中根据它来从block结构体中取到捕获的对象，如`__main_block_func_0`所示。

- 对象

{% highlight cpp %}
struct __main_block_impl_0 {
  ...
  __strong id obj;
  }
};

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->obj, (void*)src->obj, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->obj, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __main_block_desc_0 {
  ...
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
};

void main() {
  id obj = [NSMutableArray new];
	^{ [obj addObject:@1]; };
}
{% endhighlight %}

当block中使用一个对象时，block结构体同样添加了该对象，同时附上了该对象的内存修饰符`__strong`。同时描述blcok结构体的desc结构体中拥有增加了两个成员，两个函数指针，现在暂且记下，稍后笔者会解释。

跟之前的整型一样，blcok中不能修改对象，但是可以使用对象的方法，比如往数组中添加一个元素。

### 内存管理

### 小结

## lambda


## 总结

## References

[iOS与OS X多线程和内存管理](https://book.douban.com/subject/24720270/)
[https://www.mikeash.com/pyblog/friday-qa-2011-06-03-objective-c-blocks-vs-c0x-lambdas-fight.html](https://www.mikeash.com/pyblog/friday-qa-2011-06-03-objective-c-blocks-vs-c0x-lambdas-fight.html)