---
layout: post
title:  "【Runtime源码】类的propety 和 ivar"
date:   2019-01-07
excerpt:  "本文是笔者阅读Runtime源码关于类的propety 和 ivar的笔记"
tag:
- SourceCode
comments: true
---

> 本文笔者根据 Runtime 750 源码分析OC中类的propety 和 ivar的数据结构。

前面笔者介绍过了类的加载过程，不过没有详细的介绍关于 property 和 ivar，本文笔者就着重分析这两个类的组成部分，测试类如下所示:

{% highlight objective_c %}
/*****************************************************************/

@interface Person : NSObject {
    NSString *_nickName;
}
@property (nonatomic, copy) NSString *name;
@end

@implementation Person

@end

/*****************************************************************/

@interface Student: Person {
    int _score;
}
@property (nonatomic, assign) NSInteger grade;
@end

@implementation Student

@end

/*****************************************************************/
{% endhighlight %}

{% highlight cpp %}
struct class_ro_t { 
    const ivar_list_t * ivars;
}
{% endhighlight %}

其实通过之前的 `class_rw_t` 和 `class_ro_t` 的组成结构，我们可以发现 rw 中少了 ivar列表只有 property列表，而 ro 中则两者都有, 而且我们可以发现 ivars 这个属性也是 const 常量，初始化就不能修改的。

所以我们可以知道类初始化后就不可以在动态添加 ivar 了，因为此时类的大小和内存布局已经确定，但是可以通过分类动态添加property，其实就是 getter 和 setter 方法。想象一下假设可以动态添加 ivar 那么类的内存布局就改变了，之前类创建的实例则不能正常工作了，而分类中添加方法最终会通过之前说过的 `attachLists` 方法将其添加到 rw 的 可以扩容的 `methods` 方法数组中。

## property

![runtime_source_code_property_and_ivar_1]({{site.url}}/assets/images/blog/runtime_source_code_property_and_ivar_1.png)

我们看到Student 类中添加了一个 property grade，但是通过上图我们发现类中还增加了一个下划线(_)开头的ivar和两个方法，也就是这个ivar的 getter 和 setter。所以现在我们知道propety其实等于 ivar + getter + setter。

{% highlight cpp %}
typedef struct property_t *objc_property_t;

struct property_t {
    const char *name;
    const char *attributes;
};
{% endhighlight %}

再看 property 的结构，很简单的一个结构体，两个成员，attributes 用来记录property的属性，是一个字符串类型，是根据[Property Type String](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101) 和 [Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html) 两部分组成的，具体可以去官方文档。

## ivar

{% highlight cpp %}
typedef struct ivar_t *Ivar;

struct ivar_t {
    int32_t *offset;
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};
{% endhighlight %}

我们看到 ivar 的结构就稍微复杂点了，多了不少属性。

![runtime_source_code_property_and_ivar_2]({{site.url}}/assets/images/blog/runtime_source_code_property_and_ivar_2.png)

![runtime_source_code_property_and_ivar_3]({{site.url}}/assets/images/blog/runtime_source_code_property_and_ivar_3.png)

通过上面两张图我们发现:

```
Person: 
instanceStart: 8
instanceSize: 24

Student:
instanceStart: 24
instanceSize: 40
```

我们知道 NSObject 只有一个成员，就是我们之前说过的 `isa`, 所以 NSObject 的 instanceSize 为 8，所以 ro 的 instanceStart 就是 父类的 instanceSize。而且我们发现子类会拥有父类的属性，从 instanceSize 就可以看出来。

之前在类的加载中我们有说到当前类的 instanceStart 如果小于父类的 instanceSize，说明父类中新加属性，导致子类内存布局和父类重叠，所以需要调整，下面笔者来分析下调整的过程。

{% highlight cpp %}
static void moveIvars(class_ro_t *ro, uint32_t superSize) {
    uint32_t diff;

    assert(superSize > ro->instanceStart);
    diff = superSize - ro->instanceStart;

    if (ro->ivars) {
        uint32_t maxAlignment = 1;
        for (const auto& ivar : *ro->ivars) {
            if (!ivar.offset) continue;  // anonymous bitfield

            uint32_t alignment = ivar.alignment();
            if (alignment > maxAlignment) maxAlignment = alignment;
        }

        uint32_t alignMask = maxAlignment - 1;
        diff = (diff + alignMask) & ~alignMask;

        for (const auto& ivar : *ro->ivars) {
            if (!ivar.offset) continue;  // anonymous bitfield

            uint32_t oldOffset = (uint32_t)*ivar.offset;
            uint32_t newOffset = oldOffset + diff;
            *ivar.offset = newOffset;
        }
    }

    *(uint32_t *)&ro->instanceStart += diff;
    *(uint32_t *)&ro->instanceSize += diff;
}
static void reconcileInstanceVariables(Class cls, Class supercls, const class_ro_t*& ro) {
    class_rw_t *rw = cls->data();

    assert(supercls);
    assert(!cls->isMetaClass());

    const class_ro_t *super_ro = supercls->data()->ro;

    if (ro->instanceStart >= super_ro->instanceSize) {
        return;
    }
    if (ro->instanceStart < super_ro->instanceSize) {
        class_ro_t *ro_w = make_ro_writeable(rw);
        ro = rw->ro;
        moveIvars(ro_w, super_ro->instanceSize);
        gdb_objc_class_changed(cls, OBJC_CLASS_IVARS_CHANGED, ro->name);
    } 
}
{% endhighlight %}

`moveIvars` 分为以下几个步骤:

```
1. 根据父类 instanceSize 和 子类 instanceStart 差值记做 diff
2. 求出子类所有 ivar 中最大的对齐长度，记做 maxAlignment
3. 尝试将 diff 按照 maxAlignment 进行对齐
4. 更新子类所有 ivar 的 offest 成员，将原来 offset 指向的值加 diff
5. 最后依然使用 diff，更新子类 ro 的 instanceStart 和 instanceSize
```

