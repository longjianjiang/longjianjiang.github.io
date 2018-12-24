---
layout: post
title:  "【Runtime源码】类的结构"
date:   2018-12-21
excerpt:  "本文是笔者阅读Runtime源码关于类的结构的笔记"
tag:
- SourceCode
comments: true
---

> 本文笔者根据 Runtime 750 源码分析OC中类和对象的具体结构。

OC中的对象都是 `objc_object` 结构体，OC中的类都是 `objc_class` 结构体，`objc_class` 又继承自 `objc_object`，所以OC中大家都是对象。

```
typedef struct objc_class *Class;
typedef struct objc_object *id;

struct objc_object {
private:
    isa_t isa;
};

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
};
```

根据上面代码我们知道了 `id` 和 `Class` 指的是什么，也看到了两个结构体中所包含的属性。

## isa

根据👆两个结构体，我们发现OC中不管是对象还是类都有 `isa` 这个属性，下面笔者来分析这个属性。

下面是关于 isa 的几个宏:

```
SUPPORT_PACKED_ISA: 这个宏为 1 时，此时 isa 将 class 信息存储在结构体中，同时存储了其他的信息。在 iOS 和 macOS 平台下 这个宏为 1。

SUPPORT_INDEXED_ISA: 这个宏为 1 时，此时 isa 中并没有直接存储 class 信息，而是存储了一个该类在一个全局的类表中的索引，通过这个索引可以取得该类的信息。iOS 和 macOS 平台下这个宏为 0。

SUPPORT_NONPOINTER_ISA: 这个宏就是根据上面两个宏来得到的，当上述有一个为 1，该宏也为1，说明此时 isa 中不再是单纯的指针，还存储了其他的信息。
```

👇是 isa_t 声明，以及结构体各个成员的作用：

```
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
      //ISA_BITFIELD;  // defined in isa.h
      uintptr_t nonpointer        : 1;                       
      uintptr_t has_assoc         : 1; // 标记是否存在关联对象，也就是使用 `objc_setAssociatedObject` 添加的；                            
      uintptr_t has_cxx_dtor      : 1;          
      uintptr_t shiftcls          : 44; /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/ 
      uintptr_t magic             : 6; // 标记对象是否初始化完成；                              
      uintptr_t weakly_referenced : 1; // 标记对象被指向或者曾经指向一个弱引用，没有弱引用的对象可以更快释放；
      uintptr_t deallocating      : 1; // 标记对象是否正在释放内存；           
      uintptr_t has_sidetable_rc  : 1; // 如果👇存储的引用计数大于10，该字段存储进位；
      uintptr_t extra_rc          : 8  // 存储对象的引用计数，当对象引用计数为10，该字段存9；
    };
#endif
};
```

> isa_t 根据平台的不同结构体中的成员所占的位数不一样，👆摘录了 x86 的实现；

该属性是一个 union，而且我们看到 union中有一个成员 是 `Class`, 也就是我们所说的 isa 指向的类。

对象的 isa 指向 类， 类的 isa 指向 metaClass，如下图所示:

![runtime_source_code_first_1]({{site.url}}/assets/images/blog/runtime_source_code_first_1.png)

正因为这样， `objc_object` 中有以下方法来初始化 isa.

```
void initClassIsa(Class cls /*nonpointer=maybe*/);
void initInstanceIsa(Class cls, bool hasCxxDtor);
```

上面两个初始化 isa 的方法，内部都会调用 `objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) ` 进行初始化。

- nonpointer

上面这个参数用来说明isa 的存储方式，为false 的话，isa 联合中没有结构体成员；

为true 的话，主要是64位地址位数比较多，为了降低内存使用，所以 isa 中不仅仅存储类的信息，还存储了一些其他的信息，也就是我们先前看到的联合中的结构体。

- hasCxxDtor

这个参数用来说明当前对象中有没有C++析构函数，为了兼容 .mm，如果是 .mm 需要做一些额外的析构工作。

```
define ISA_MAGIC_VALUE 0x001d800000000001ULL

inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    if (!nonpointer) {
        isa.cls = cls;
    } else {
        isa_t newisa(0);

        newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
        isa = newisa;
    }
}
```

> 上述代码去除了一些 assert 和注释，同时假定nonpointer 为 true，省略了 SUPPORT_INDEXED_ISA 为 1 的情况。

上述代码就是内部用来初始化 isa 的方法，我们看到如果 nonpointer 为false，直接赋值 cls 存储类的信息。

当nonpointer 为 true 时，先新建一个 newisa，同时将各位赋值为0，开始初始化 isa 联合中的几个成员。

- bits

根据 ISA_MAGIC_VALUE 64位的二进制数据 以及 结构体中各个成员所占的位数，我们可以知道初始化了 `nonpointer` 和 `magic` 两个成员。

- has_cxx_dtor

继续初始化 联合结构体中 `has_cxx_dtor` 成员。

- shiftcls

这个成员就是当 nonpointer 为 true 时，结构体中44位用来存储 cls 信息的。(从命名也可以看出来，shift cls)

> 赋值的时候我们将传递过来的参数 cls 右移了 3位，说明 cls 的低三位默认是0。

因此 此时 nonpointer 为 true，cls 信息存储在了结构体中，所以 `objc_object` 提供了一个获取 cls 信息的函数。

- ISA()

```
define ISA_MASK        0x00007ffffffffff8ULL

inline Class 
objc_object::ISA() 
{
    return (Class)(isa.bits & ISA_MASK);
}
```

> 上述代码去除了一些 assert 和 注释，同时去除了 SUPPORT_INDEXED_ISA 为 1 的分支。

简单的通过 & 运算取得结构体中44位存储 cls 信息的将其转成 Class 类型返回。

将 newisa 赋值给 isa 成员变量，至次初始化 isa 结束。