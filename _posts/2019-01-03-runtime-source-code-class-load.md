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

{% highlight objective_c %}
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
{% endhighlight %}

## realizeClass

这个方法就是对类进行第一次加载的，主要作用是为 `class_rw_t` 结构体成员分配空间，同时初始化各个成员返回类的真正结构。下面笔者就一步一步的展示这一过程。

{% highlight cpp %}
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
{% endhighlight %}

开始定义了需要用到的变量，然后做了三步判断，保证 cls 参数的有效性。

- 判断cls是否为空，否则返回 nil；
- 判断cls是否已经初始化过，通过之前所说的 `class_rw_t` 的 bits 与标记位获得，否则直接返回 cls；
- 断言当前cls没有初始化，通过 `remapClass` 方法从一个全局的 `NXMapTable` 中找 key 为 cls 是否存在 value，不存在value 或者全局的 `NXMapTable` 为空，则直接返回 cls；

{% highlight cpp %}
static Class realizeClass(Class cls) {
    ...

    ro = (const class_ro_t *)cls->data();
     // Normal class. Allocate writeable class data.
    rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
    rw->ro = ro;
    rw->flags = RW_REALIZED|RW_REALIZING;
    cls->setData(rw);

    isMeta = ro->flags & RO_META;

    rw->version = isMeta ? 7 : 0;  // old runtime went up to 6
    ...
}
{% endhighlight %}

接着通过 `objc_class` data() 方法将原本是 `class_rw_t` 强转成 `class_ro_t`。也就是说类刚初始化时 data() 中 FAST_DATA_MASK 标记位存储的其实是 `class_ro_t`, 所以强转才能成功。

根据 ro， 为 rw 分配地址空间，将 ro 赋值给 rw，设置 rw 为已初始化，将 rw 设置到 `objc_class` 的 bits 中，根据 ro flags 得到是否是 metaClass，赋值给rw的version。我们了解了结构后，这一步还是比较容易理解的。

![runtime_source_code_class_load_1]({{site.url}}/assets/images/blog/runtime_source_code_class_load_1.png)

通过截图我们看到 ro 中 method、ivars、property的列表都是有值的，而且笔者通过 lldb 打印出了当前类在编译期间中的所有方法。

不过我们发现多了一个 `.cxx_destruct` 的方法，这个方法是用来销毁实例变量的，只有类中有实例变量，不管是不是通过 propety 生成的，具体可以参考[.cxx_destruct](http://blog.sunnyxx.com/2014/04/02/objc_dig_arc_dealloc/)

{% highlight cpp %}
static Class realizeClass(Class cls) {
    ...

    supercls = realizeClass(remapClass(cls->superclass));
    metacls = realizeClass(remapClass(cls->ISA()));

    ...
}
{% endhighlight %}

这一步获得 superCls 和 metaCls，依然通过 `realizeClass` 方法，尝试给父类和原类初始化。

然后 `realizeClass` 会尝试禁用 non-pointer isa的结构，这里笔者就不贴代码了，一般情况还是 non-pointer isa 的结构。

{% highlight cpp %}
static Class realizeClass(Class cls) {
    ...

    cls->superclass = supercls;
    cls->initClassIsa(metacls);

    ...
}
{% endhighlight %}

根据之前获得的 superCls、metaCls，设置 cls 的父类，设置 cls 的原类。关于初始化类的isa，具体可以参考笔者之前的[isa笔记](http://www.longjianjiang.com/runtime-source-code-class-isa/)。

{% highlight cpp %}
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
static Class realizeClass(Class cls) {
    ...

    if (supercls  &&  !isMeta) reconcileInstanceVariables(cls, supercls, ro);

    ...
}
{% endhighlight %}

这里判断当前类的 instanceStart 如果小于父类的 instanceSize，说明父类中新加属性，导致子类内存布局和父类重叠，所以需要调整，具体调整实现在 `moveIvars`, 具体实现笔者会在关于 ivar 中详细叙述。

{% highlight cpp %}
static Class _firstRealizedClass = nil;

static void addRootClass(Class cls) {
    assert(cls->isRealized());
    cls->data()->nextSiblingClass = _firstRealizedClass;
    _firstRealizedClass = cls;
}

static void addSubclass(Class supercls, Class subcls) {
    if (supercls  &&  subcls) {
        assert(supercls->isRealized());
        assert(subcls->isRealized());
        subcls->data()->nextSiblingClass = supercls->data()->firstSubclass;
        supercls->data()->firstSubclass = subcls;

        if (supercls->hasCxxCtor()) {
            subcls->setHasCxxCtor();
        }

        if (supercls->hasCxxDtor()) {
            subcls->setHasCxxDtor();
        }

        if (supercls->hasCustomRR()) {
            subcls->setHasCustomRR(true);
        }

        if (supercls->hasCustomAWZ()) {
            subcls->setHasCustomAWZ(true);
        }

        // Special case: instancesRequireRawIsa does not propagate 
        // from root class to root metaclass
        if (supercls->instancesRequireRawIsa()  &&  supercls->superclass) {
            subcls->setInstancesRequireRawIsa(true);
        }
    }
}

static Class realizeClass(Class cls) {
    ...

    cls->setInstanceSize(ro->instanceSize);

    if (ro->flags & RO_HAS_CXX_STRUCTORS) {
        cls->setHasCxxDtor();
        if (! (ro->flags & RO_HAS_CXX_DTOR_ONLY)) {
            cls->setHasCxxCtor();
        }
    }

    if (supercls) {
        addSubclass(supercls, cls);
    } else {
        addRootClass(cls);
    }

    ...
}
{% endhighlight %}

这里是将 ro 的信息复制到 rw 中，首先赋值 instanceSize，然后尝试设置是否有C++构造函数和析构函数。

根据是否是基类，分别调用 addSubclass, addRootClass。addSubclass 中给 `objc_class` 的 `nextSiblingClass` 和 `firstSubclass` 进行赋值。然后传递父类的属性给子类。addRootClass 则更加简单用 `_firstRealizedClass` 也就是 nil 赋值给 NSObject 的 `nextSiblingClass`。

## methodizeClass

{% highlight cpp %}
static Class realizeClass(Class cls) {
    ...

    methodizeClass(cls);

    return cls;

    ...
}
{% endhighlight %}

经过前面的处理，到了一个关键的步骤就是 `methodizeClass`, 这个方法内部会更新cls 的 `class_rw_t` 方法列表、协议列表、属性列表。

{% highlight cpp %}
static void methodizeClass(Class cls) {
    bool isMeta = cls->isMetaClass();
    auto rw = cls->data();
    auto ro = rw->ro;

    method_list_t *list = ro->baseMethods();
    if (list) {
        prepareMethodLists(cls, &list, 1, YES, isBundleClass(cls));
        rw->methods.attachLists(&list, 1);
    }
    property_list_t *proplist = ro->baseProperties;
    if (proplist) {
        rw->properties.attachLists(&proplist, 1);
    }

    protocol_list_t *protolist = ro->baseProtocols;
    if (protolist) {
        rw->protocols.attachLists(&protolist, 1);
    }

    ...
}
{% endhighlight %}

可以看到这里通过 cls 获得 rw，ro，将ro 里编译期存在的方法，协议，属性加到 rw 中，使用的就是之前说过的 `list_array_tt` 的`attachLists` 方法，具体实现可以参考[class_data_bits](http://www.longjianjiang.com/runtime-source-code-class-data-bits/)。

{% highlight cpp %}
static NXMapTable *namedSelectors;

static SEL __sel_registerName(const char *name, bool shouldLock, bool copy) {
    SEL result = 0;

    if (!name) return (SEL)0;

    result = search_builtins(name);
    if (result) return result;
    
    if (namedSelectors) {
        result = (SEL)NXMapGet(namedSelectors, name);
    }
    if (result) return result;

    if (!namedSelectors) {
        namedSelectors = NXCreateMapTable(NXStrValueMapPrototype, 
                                          (unsigned)SelrefCount);
    }
    if (!result) {
        result = sel_alloc(name, copy);
        NXMapInsert(namedSelectors, sel_getName(result), result);
    }

    return result;
}

SEL sel_registerNameNoLock(const char *name, bool copy) {
    return __sel_registerName(name, 0, copy);  // NO lock, maybe copy
}

static void 
fixupMethodList(method_list_t *mlist, bool bundleCopy, bool sort) {
    assert(!mlist->isFixedUp());
    {
        for (auto& meth : *mlist) {
            const char *name = sel_cname(meth.name);
            meth.name = sel_registerNameNoLock(name, bundleCopy);
        }
    }

    if (sort) {
        method_t::SortBySELAddress sorter;
        std::stable_sort(mlist->begin(), mlist->end(), sorter);
    }
    
    mlist->setFixedUp();
}

static void 
prepareMethodLists(Class cls, method_list_t **addedLists, int addedCount, 
                   bool baseMethods, bool methodsFromBundle) {
    if (addedCount == 0) return;

    if (baseMethods) {
        assert(!scanForCustomRR  &&  !scanForCustomAWZ);
    }

    for (int i = 0; i < addedCount; i++) {
        method_list_t *mlist = addedLists[i];
        assert(mlist);

        if (!mlist->isFixedUp()) {
            fixupMethodList(mlist, methodsFromBundle, true/*sort*/);
        }
    }
}
{% endhighlight %}

> prepareMethodLists 方法中省略了关于 内存管理方法 和 allocWithZone 方法的设置；

不过在添加方法到 rw 中时，多了一个 prepare 的步骤。这个方法主要是调用 `fixupMethodList`, 这个 fixup 方法主要将方法的 SEL 注册到一个全局的 `NXMapTable` 中，完成后标记 `method_list_t` fixup 完成。

{% highlight cpp %}
static Class realizeClass(Class cls) {
    ...

    // Root classes get bonus method implementations if they don't have 
    // them already. These apply before category replacements.
    if (cls->isRootMetaclass()) {
        // root metaclass
        addMethod(cls, SEL_initialize, (IMP)&objc_noop_imp, "", NO);
    }

    // Attach categories.
    category_list *cats = unattachedCategoriesForClass(cls, true /*realizing*/);
    attachCategories(cls, cats, false /*don't flush caches*/);
    
    if (cats) free(cats);

    ...
}
{% endhighlight %}

这里首先判断当前 cls 是不是 rootMetaClass，也就是NSObject 的 metaClass，如果是给这个类增加了一个方法，`addMethod` 的具体实现这里不展开。

接着会尝试更新类的协议列表，具体获取和添加的过程具体实现这里不展开。

至此，第一次初始化类的工作全部完成。