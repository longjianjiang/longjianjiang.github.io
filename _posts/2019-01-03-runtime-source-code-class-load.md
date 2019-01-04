---
layout: post
title:  "【Runtime源码】类的加载过程"
date:   2019-01-03
excerpt:  "本文是笔者阅读Runtime源码关于类的加载的笔记"
tag:
- SourceCode
comments: true
---

> 本文笔者根据 Runtime 750 源码分析OC中类加载的具体过程。

在前面笔者介绍了类的结构(isa, class_data_bits_t), 本文笔者介绍类的加载过程。测试类如下:

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

## realizeClass

这个方法就是对类进行第一次加载的，主要作用是为 `class_rw_t` 结构体成员分配空间，同时初始化各个成员返回类的真正结构。下面笔者就一步一步的展示这一过程。

```
static Class realizeClass(Class cls) {
    const class_ro_t *ro;
    class_rw_t *rw;
    Class supercls;
    Class metacls;
    bool isMeta;

    if (!cls) return nil;
    if (cls->isRealized()) return cls;
    assert(cls == remapClass(cls));

    ...
}

static Class remapClass(Class cls) {
    Class c2;

    if (!cls) return nil;

    NXMapTable *map = remappedClasses(NO);
    if (!map  ||  NXMapMember(map, cls, (void**)&c2) == NX_MAPNOTAKEY) {
        return cls;
    } else {
        return c2;
    }
}
```

开始定义了需要用到的变量，然后做了三步判断，保证 cls 参数的有效性。

- 判断cls是否为空，否则返回 nil；
- 判断cls是否已经初始化过，通过之前所说的 `class_rw_t` 的 bits 与标记位获得，否则直接返回 cls；
- 断言当前cls没有初始化，通过 `remapClass` 方法从一个全局的 `NXMapTable` 中找 key 为 cls 是否存在 value，不存在value 或者全局的 `NXMapTable` 为空，则直接返回 cls；

```
static Class realizeClass(Class cls) {
    ...

    ro = (const class_ro_t *)cls->data();
     // Normal class. Allocate writeable class data.
    rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
    rw->ro = ro;
    rw->flags = RW_REALIZED|RW_REALIZING;
    cls->setData(rw);

    ...
}
```

接着通过 `objc_class` data() 方法将原本是 `class_rw_t` 强转成 `class_ro_t`。也就是说类刚初始化时 data() 中 FAST_DATA_MASK 标记位存储的其实是 `class_ro_t`, 所以强转才能成功。

根据 ro， 为 rw 分配地址空间，将 ro 赋值给 rw，设置 rw 为已初始化，将 rw 设置到 `objc_class` 的 bits 中，我们了解了结构后，这一步还是比较容易理解的。

![runtime_source_code_class_load_1]({{site.url}}/assets/images/blog/runtime_source_code_class_load_1.png)

通过截图我们看到 ro 中 method、ivars、property的列表都是有值的，而且笔者通过 lldb 打印出了当前类在编译期间中的所有方法。

不过我们发现多了一个 `.cxx_destruct` 的方法，这个方法是用来销毁实例变量的，只有类中有实例变量，不管是不是通过 propety 生成的，具体可以参考[.cxx_destruct](http://blog.sunnyxx.com/2014/04/02/objc_dig_arc_dealloc/)

