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

{% highlight cpp %}
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
};
{% endhighlight %}

之前笔者分析过 `objc_class` 结构体中继承自 `objc_object` 的 `isa` 成员，本文笔者来分析 `class_data_bits_t` 成员的作用与结构。

## class_data_bits_t

{% highlight cpp %}
struct class_data_bits_t {
    // Values are the FAST_ flags above.
    uintptr_t bits;
}
{% endhighlight %}

我们可以看到 `class_data_bits_t` 结构体只有一个 64位的 bits来存储相关内容，而且注释也写了这些内容可以通过 FAST_ 开头的标记位按位与得到。

这些FAST_ 开头的标记位根据平台的不同，个数和位置都是不一样的，具体可以去 `objc-runtime-new.h` 中查看。

因为内容存储在64位中的每一位，所以这个结构体中主要操作就是围绕着其唯一的成员 `bits` 进行 二进制位运算，实现更新删除添加操作。

结构体中有以下方法用来操作 `bits`, 实现就是普通的位运算操作，这里笔者就不贴实现代码了，具体可以去`objc-runtime-new.h` 中查看。

{% highlight cpp %}
bool getBit(uintptr_t bit)
void setBits(uintptr_t set) 
void clearBits(uintptr_t clear) 
{% endhighlight %}

## class_rw_t & class_ro_t

{% highlight cpp %}
class_rw_t* data() {
    return (class_rw_t *)(bits & FAST_DATA_MASK);
}
{% endhighlight %}

前面说到 `class_data_bits_t`, 其中一个重要的标记位 `FAST_DATA_MASK`, 用来存储 `class_rw_t` 的指针。

{% highlight cpp %}
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
{% endhighlight %}

{% highlight cpp %}
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
{% endhighlight %}

我们看到 `class_rw_t` 中含有 `class_ro_t` 结构体。`class_ro_t` 我们看到有关于method，protocol，ivar，property的信息，其中除了protocol其他三个都是继承自 `entsize_list_tt` 类型的结构体。这个结构体中存储了类在编译时就确定的信息，也就是前面所说的四种信息。

{% highlight cpp %}
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
{% endhighlight %}

`entsize_list_tt` 定义了一种类似数组的顺序存储的结构，内置了 Random Access Iterator, 提供了根据索引获取存储Element的方法 `get(uint32_t idx)`。

`class_ro_t` 的 flags 存储了一些类在编译时期就确定的信息，也是以标记位的形式来存储，这些标记位以 `RO_` 来头，如下所示，这里只取了部分:

{% highlight cpp %}
// class is a metaclass
#define RO_META               (1<<0)
// class is a root class
#define RO_ROOT               (1<<1)
// class has .cxx_construct/destruct implementations
#define RO_HAS_CXX_STRUCTORS  (1<<2)
{% endhighlight %}

`class_ro_t` 的 `instanceStart` 和 `instanceSize` 则和 non-fragile ivars 有关，关于 non-fragile ivars 可以参考笔者之前的[Runtime学习笔记](http://www.longjianjiang.com/runtime/)。编译时期这两个属性就已经被赋值, 初始化类时会判断当前类的 `instanceStart` 如果小于父类的 `instanceStart` 就会进行调整，这样当 `NSObject`等基类中增加了属性，我们编写的代码不用重新编译，Runtime会自动调整，这就是所谓的 non-fragile ivars 特性，具体实现笔者会在关于 ivar 中详细叙述。

下面我们来看 `class_rw_t` 结构体同样存在关于方法，属性，协议的属性，他们的类型都继承自 `list_array_tt`。类刚初始化的时候，这些属性的值都是空。

{% highlight cpp %}
template <typename Element, typename List>
class list_array_tt {
    struct array_t {
        uint32_t count;
        List* lists[0];

        static size_t byteSize(uint32_t count) {
            return sizeof(array_t) + count*sizeof(lists[0]);
        }
        size_t byteSize() {
            return byteSize(count);
        }
    };

    // iterator implementation at objc-runtime-new.h

private:
    union {
        List* list;
        uintptr_t arrayAndFlag;
    };

public:
    void attachLists(List* const * addedLists, uint32_t addedCount) { ... }
};
{% endhighlight %}

上述就是 `list_array_tt` 的大致结构，省略了一些代码实现。这个结构内部创建了一个类似数组的叫 `array_t` 的结构体，内部有个 `lists` 数组 存储 `List *` 指针类型。如果 List 的个数大于1了就需要使用这个`lists`了，否则使用联合的 `list` 成员。

一个重要的方法就是 `attachLists`， 就是在该方法中将 `class_ro_t` 中方法，属性，协议等信息加入到 `class_rw_t` 的对应 `list_array_tt` 类型的属性中的。同时通过分类给类增加方法也是通过该方法加入的，具体添加过程笔者这里不展开。

通过 `attachLists` 方法内部会将新添加的List移动到最前面，将之前的元素移动到新添加List的尾部。

`class_rw_t` 的 flags 也存储了一些类相关的信息，也是以标记位的形式来存储，这些标记位以 `RW_` 来头，这些信息不是编译期间确定的，这里只取了部分:

{% highlight cpp %}
#define RW_REALIZED           (1<<31)
// class is unresolved future class
#define RW_FUTURE             (1<<30)
// class is initialized
#define RW_INITIALIZED        (1<<29)
{% endhighlight %}

## 最后

通过上面分析，我们知道了 OC 类中一个重要的成员 bits。同时知道了 bits 里一个重要的 `data()` 返回了表示类结构的 `class_rw_t` 结构体，进而发现上述结构体中还存在一个 `class_ro_t` 结构体存储了一些编译期间确定的信息。同时介绍了两种内部集合类型 `list_array_tt`, `entsize_list_tt`。
