---
layout: post
title:  "【Runtime源码】类的结构(class_data_bits_t)"
date:   2018-12-29
excerpt:  "本文是笔者阅读Runtime源码关于class_data_bits_t的笔记"
tag:
- SourceCode
comments: true
---

> 本文笔者根据 Runtime 750 源码分析OC中类 class_data_bits_t 的结构。

```
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
};
```

之前笔者分析过 `objc_class` 结构体中继承自 `objc_object` 的 `isa` 成员，本文笔者来分析 `class_data_bits_t` 成员的作用与结构。

## class_data_bits_t

```
struct class_data_bits_t {
    // Values are the FAST_ flags above.
    uintptr_t bits;
}
```

我们可以看到 `class_data_bits_t` 结构体只有一个 64位的 bits来存储相关内容，而且注释也写了这些内容可以通过 FAST_ 开头的标记位按位与得到，不过笔者目前的runtime 版本中还有 RW_ 开头的标记位。

这些FAST_ 开头的标记位根据平台的不同，个数和位置都是不一样的，具体可以去 `objc-runtime-new.h` 中查看。

因为内容存储在64位中的每一位，所以这个结构体中主要操作就是围绕着其唯一的成员 `bits` 进行 二进制位运算，实现更新删除添加操作。

结构体中有以下方法用来操作 `bits`, 实现就是普通的位运算操作，这里笔者就不贴实现代码了，具体可以去`objc-runtime-new.h` 中查看。

```
bool getBit(uintptr_t bit)
void setBits(uintptr_t set) 
void clearBits(uintptr_t clear) 
```

## class_rw_t & class_ro_t

```
class_rw_t* data() {
    return (class_rw_t *)(bits & FAST_DATA_MASK);
}
```

前面说到 `class_data_bits_t`, 其中一个重要的标记位 `FAST_DATA_MASK`, 用来存储 `class_rw_t` 的指针。

```
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
}
```

```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
```

我们看到 `class_rw_t` 中含有 `class_ro_t` 结构体。`class_ro_t` 我们看到有关于method，protocol，ivar，property的信息，其中除了protocol其他三个都是继承自 `entsize_list_tt` 类型的结构体。

