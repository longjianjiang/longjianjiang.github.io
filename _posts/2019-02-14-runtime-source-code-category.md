---
layout: post
title:  "【Runtime源码】类的分类"
date:   2019-02-14
excerpt:  "本文是笔者阅读Runtime源码关于类的分类的笔记"
tag:
- Runtime
comments: true
---

> 本文笔者根据 Runtime 750 源码分析OC中类分类的数据结构以及关联对象的实现。

本文测试类如下所示:

{% highlight objective_c %}
/*****************************************************************/

@interface Person : NSObject {
    @public
    NSString *_nickName;
}

- (void)say;
@end

@implementation Person
- (void)say {
    NSLog(@"hello world");
}

@end
/*****************************************************************/

@interface Person (Extension)
- (void)personAddMethod;
@end

@implementation Person (Extension)
- (void)personAddMethod {
    NSLog(@"personAddMethod");
}
@end

@interface NSString (Extension)
- (void)stringAddMethod;
@end

@implementation NSString (Extension)
- (void)stringAddMethod {
    NSLog(@"stringAddMethod");
}
@end

@interface NSObject (Extension)
- (void)objectAddMethod;
@end

@implementation NSObject (Extension)
- (void)objectAddMethod {
    NSLog(@"objectAddMethod");
}
@end
/*****************************************************************/
{% endhighlight %}

# Category Method

分类允许我们动态的给系统类或者自定义类添加方法，属性，同时分类中也可以遵守协议。

{% highlight cpp %}
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    struct property_list_t *_classProperties;
};
{% endhighlight %}

分类在Runtime中同样使用结构体来表示，通过结构体中的成员也证实了之前所说的分类具有的功能。

下面笔者来查看Runtime运行时将分类中方法添加到类中的实现过程。

在测试代码中，笔者分别为自定义类`Person`和系统类`NSString`,`NSObject`添加了分类，分类中只有一个方法，当分类中添加的方法名和类中已有方法重复，编译器会报警告，此时如果调用该方法，执行的是分类中的实现。

![runtime_source_code_category_1]({{site.url}}/assets/images/blog/runtime_source_code_category_1.png)

{% highlight cpp %}
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches) {
    if (!cats) return;
    bool isMeta = cls->isMetaClass();

    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
{% endhighlight %}

`attachCategories`就是实际将分类中方法，属性，协议添加到类的rw对应的列表中，实现也比较直观，分三步：

1.新建方法列表，属性列表，协议列表；

2.遍历协议列表`cats`，获取每个协议中的方法列表，属性列表，协议列表，添加到第一步创建的列表中；

3.取到类rw中方法列表，属性列表，协议列表，使用`attachLists`将第一步新建的列表，添加到rw中；

知道了如何添加分类中的内容到类中后，下面一个问题就是何时调用`attachCategories`，笔者在`attachCategories`打了条件断点，当cls为Person，NSString，NSObject时会激活。

![runtime_source_code_category_2]({{site.url}}/assets/images/blog/runtime_source_code_category_2.png)

![runtime_source_code_category_3]({{site.url}}/assets/images/blog/runtime_source_code_category_3.png)

![runtime_source_code_category_4]({{site.url}}/assets/images/blog/runtime_source_code_category_4.png)

NSObject走到该方法多次，Person和NSString走到该方法一次。