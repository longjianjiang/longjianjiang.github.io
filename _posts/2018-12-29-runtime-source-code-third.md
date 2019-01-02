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

## 测试代码

```
@interface Person : NSObject {
    NSString *_nickName;
}

@property (nonatomic, copy) NSString *name;

- (void)say;
- (void)write;
@end

@implementation Person
- (void)say {
    NSLog(@"hello world");
}

- (void)write {
    NSLog(@"write some letter");
}
@end
```

笔者以👆代码来进行测试。

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

我们看到 `class_rw_t` 中含有 `class_ro_t` 结构体。`class_ro_t` 我们看到有关于method，protocol，ivar，property的信息，其中除了protocol其他三个都是继承自 `entsize_list_tt` 类型的结构体。这个结构体中存储了类在编译时就确定的信息，也就是前面所说的四种信息。

```
template <typename Element, typename List, uint32_t FlagMask>
struct entsize_list_tt {
    uint32_t entsizeAndFlags;
    uint32_t count;
    Element first;

    uint32_t entsize() const {
        return entsizeAndFlags & ~FlagMask;
    }

    Element& getOrEnd(uint32_t i) const { 
        assert(i <= count);
        return *(Element *)((uint8_t *)&first + i*entsize()); 
    }
    Element& get(uint32_t i) const { 
        assert(i < count);
        return getOrEnd(i);
    }
    
    // iterator implementation at objc-runtime-new.h
};
```

`entsize_list_tt` 定义了一种类似数组的顺序存储的结构，内置了 Random Access Iterator, 提供了根据索引获取存储Element的方法 `get(uint32_t idx)`。

`class_ro_t` 的 flags 存储了一些类在编译时期就确定的信息，也是以标记位的形式来存储，这些标记位以 `RO_` 来头，如下所示，这里只取了部分:

```
// class is a metaclass
#define RO_META               (1<<0)
// class is a root class
#define RO_ROOT               (1<<1)
// class has .cxx_construct/destruct implementations
#define RO_HAS_CXX_STRUCTORS  (1<<2)
```

`class_ro_t` 的 `instanceStart` 和 `instanceSize` 则和 non-fragile ivars 有关，关于 non-fragile ivars 可以参考笔者之前的[Runtime学习笔记](http://www.longjianjiang.com/runtime/)。


