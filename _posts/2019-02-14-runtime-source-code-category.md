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

NSObject走到该方法多次，Person和NSString走到该方法一次。通过上图我们可以知道加载分类在`map_images`通过 `remethodizeClass`和`load_images`通过 `methodizeClass` 调用`attachCategories`。

继续看Person类的情况，我们发现 `cats` 为NULL，也就是没有分类，那分类中的内容如何添加到类中呢？经过第二次调试发现，分类中的方法`personAddMethod`在编译期间已经添加到类的`ro`的方法列表中，这也解释了为什么Person调用`attachCategories`时 `cats` 为NULL。

![runtime_source_code_category_5]({{site.url}}/assets/images/blog/runtime_source_code_category_5.png)

# Category Property

笔者在[类的propety 和 ivar](http://www.longjianjiang.com/runtime-source-code-property-and-ivar/)中说过`@property`其实为我们生成了一个带下划线的实例变量和getter和setter方法。分类中同样可以使用`@property`声明一个属性，但是分类并不会帮我们自动生成一个带下划线的实例变量和getter和setter方法，所以此时就需要使用关联对象来完成分类中属性的定义。

{% highlight objective_c %}
@interface Person (Extension)
@property(nonatomic, copy) NSString *personAddProperty;
@end

@implementation Person (Extension)
- (NSString *)personAddProperty {
    return objc_getAssociatedObject(self, _cmd);
}

- (void)setPersonAddProperty:(NSString *)personAddProperty {
    objc_setAssociatedObject(self, @selector(personAddProperty), personAddProperty, OBJC_ASSOCIATION_COPY);
}
@end
{% endhighlight %}

上面就是一个简单的关联对象的例子，重点是Runtime提供的两个方法，笔者下面来看下其实现:

{% highlight cpp %}
void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    // retain the new value (if any) outside the lock.
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {
            // break any existing association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } else {
            // setting the association to nil breaks the association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);
                }
            }
        }
    }
    // release the old value (outside of the lock).
    if (old_association.hasValue()) ReleaseValue()(old_association);
}
{% endhighlight %}

`objc_setAssociatedObject`的实现就是简单的调用上面的方法，在说`_object_set_associative_reference`之前必须得看下关联对象中定义的几个类。

- ObjcAssociation

{% highlight cpp %}
class ObjcAssociation {
    uintptr_t _policy;
    id _value;
public:
    ObjcAssociation(uintptr_t policy, id value) : _policy(policy), _value(value) {}
    ObjcAssociation() : _policy(0), _value(nil) {}

    uintptr_t policy() const { return _policy; }
    id value() const { return _value; }
    
    bool hasValue() { return _value != nil; }
};
{% endhighlight %}

`ObjcAssociation` 就是内部用来存储关联对象的类。

- ObjectAssociationMap

{% highlight cpp %}
typedef ObjcAllocator<std::pair<void * const, ObjcAssociation> > ObjectAssociationMapAllocator;
class ObjectAssociationMap : public std::map<void *, ObjcAssociation, ObjectPointerLess, ObjectAssociationMapAllocator> {
public:
    void *operator new(size_t n) { return ::malloc(n); }
    void operator delete(void *ptr) { ::free(ptr); }
};
{% endhighlight %}

`ObjectAssociationMap` 其实是一个 `map`, 存储着key和ObjcAssociation的键值对。

- AssociationsHashMap

{% highlight cpp %}
typedef ObjcAllocator<std::pair<const disguised_ptr_t, ObjectAssociationMap*> > AssociationsHashMapAllocator;
class AssociationsHashMap : public unordered_map<disguised_ptr_t, ObjectAssociationMap *, DisguisedPointerHash, DisguisedPointerEqual, AssociationsHashMapAllocator> {
public:
    void *operator new(size_t n) { return ::malloc(n); }
    void operator delete(void *ptr) { ::free(ptr); }
};
{% endhighlight %}

`AssociationsHashMap` 其实是一个 `unordered_map`, 存储着对象指针和 `ObjectAssociationMap *`的键值对，也就是存储了对象的所有key和ObjcAssociation的键值对。

- AssociationsManager

{% highlight cpp %}
spinlock_t AssociationsManagerLock;

class AssociationsManager {
    // associative references: object pointer -> PtrPtrHashMap.
    static AssociationsHashMap *_map;
public:
    AssociationsManager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }
    
    AssociationsHashMap &associations() {
        if (_map == NULL)
            _map = new AssociationsHashMap();
        return *_map;
    }
};
{% endhighlight %}

`AssociationsManager` 则是一个管理类，有一个静态 `AssociationsHashMap *` 变量存储着运行时所有对象的关联对象。

上述四个类的结构大致如下所示:

```
AssociationsManager->_map
      |
AssociationsHashMap  => disguised_ptr_t : ObjectAssociationMap
      |
ObjectAssociationMap => void * : ObjcAssociation
      |
ObjcAssociation     =>  value & OBJC_ASSOCIATION_COPY
```


使用了关联对象来模拟实例变量，因为类的内存布局在编译期间就确定，运行时不允许添加额外的实例变量。


