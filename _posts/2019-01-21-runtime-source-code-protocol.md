---
layout: post
title:  "【Runtime源码】类的协议"
date:   2019-01-21
excerpt:  "本文是笔者阅读Runtime源码关于类协议的笔记"
tag:
- Runtime
comments: true
---

> 本文笔者根据 Runtime 750 源码分析OC中类协议的数据结构。

前面笔者介绍过了类的加载过程，不过没有详细的介绍 protocol，本文笔者就着重分析类的协议相关内容，测试类如下所示:

{% highlight objective_c %}
/*****************************************************************/

@protocol GenderProtocol <NSObject>
@optional
- (NSString *)gender;
@end

@protocol RemarkProtocol <NSObject>
@optional
- (NSUInteger)score;
@end

@protocol HeightProtocol <NSObject>
@required
- (CGFloat)height;
@end

/*****************************************************************/

@interface Person : NSObject<GenderProtocol, HeightProtocol> {
    @public
    NSString *_nickName;
}

- (void)say;
@end

@implementation Person
- (void)say {
    NSLog(@"hello world");
}

- (CGFloat)height {
    return 1.71;
}

- (NSString *)gender {
    return @"unknown";
}
@end

/*****************************************************************/

@interface Student: Person<RemarkProtocol>
@property (nonatomic, assign) NSInteger grade;

- (void)eat;
@end

@implementation Student

- (void)eat {
    NSLog(@"have dinner");
}

@end

/*****************************************************************/
{% endhighlight %}

## Protocol

{% highlight cpp %}
typedef uintptr_t protocol_ref_t;  // protocol_t *, but unremapped

struct protocol_t : objc_object {
    const char *mangledName;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
    // Fields below this point are not always present on disk.
    const char **_extendedMethodTypes;
    const char *_demangledName;
    property_list_t *_classProperties;

    // 省略一些方法
};
{% endhighlight %}

{% highlight cpp %}
struct protocol_list_t {
    // count is 64-bit by accident. 
    uintptr_t count;
    protocol_ref_t list[0]; // variable-size

    size_t byteSize() const {
        return sizeof(*this) + count*sizeof(list[0]);
    }

    protocol_list_t *duplicate() const {
        return (protocol_list_t *)memdup(this, this->byteSize());
    }

    typedef protocol_ref_t* iterator;
    typedef const protocol_ref_t* const_iterator;

    const_iterator begin() const {
        return list;
    }
    iterator begin() {
        return list;
    }
    const_iterator end() const {
        return list + count;
    }
    iterator end() {
        return list + count;
    }
};
{% endhighlight %}

`protocol_list_t` 一个用来存储协议的列表，相比 `ro` 中method，property，ivar使用的 `entsize_list_tt` 结构体简单许多。

可以看到 `protocol_t` 也继承自 `objc_object`，所以协议也是一个对象。下面就是协议中包含的属性，很多我们看了也能知道是我们平时写在协议中的哪部分，还有一些则是陌生的，下面笔者来介绍下协议中的属性:

- instanceMethods, classMethods: 协议中的对象方法和类方法列表
- optionalInstanceMethods, optionalClassMethods: 协议中可选的对象方法和类方法列表
- instanceProperties, _classProperties: 协议中对象property和类property(class关键字)
- size: protocol_t 类型的大小
- flags: 32位标记位，最后2位用来存储是否 `fixedup`
- protocols: 继承的其他协议
- _extendedMethodTypes: 协议中方法类型字符串数组
- mangledName, _demangledName: 这两个属性关于命名重整。

> C++有所谓命名重整(name mangling)，主要是为了支持函数重载，编译时会根据函数的参数列表的类型生成一个独一无二的名字，防止同名函数的出现，具体可以参考[name mangling](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_73/rzarg/name_mangling.htm)

![runtime_source_code_protocol_1]({{site.url}}/assets/images/blog/runtime_source_code_protocol_1.png)
![runtime_source_code_protocol_2]({{site.url}}/assets/images/blog/runtime_source_code_protocol_2.png)

根据上面两张图可以看到协议中的方法在编译期间就已经加入了`ro`的方法列表中，`ro`的协议列表也存储着当前类遵守的协议。

> 我们可以看到`ro`的协议列表并没有包含父类遵守的协议，因为方法寻找会按继承链去寻找，父类的`ro`的方法列表会添加协议中的方法，所以就没有必要在子类中显示父类的协议了。

![runtime_source_code_protocol_3]({{site.url}}/assets/images/blog/runtime_source_code_protocol_3.png)

还有一个有趣的地方是，一旦某个类遵守了协议后，`ro`的属性列表就会多出四个，这四个就是 `NSObject` 协议中定义的四个属性。

## fixupProtocol

{% highlight objective_c %}
Student *s = [[Student alloc] init];
[s score];
{% endhighlight %}

![runtime_source_code_protocol_4]({{site.url}}/assets/images/blog/runtime_source_code_protocol_4.png)

笔者发现如果对象调用了类遵守协议的可选方法没有实现时，转发过程中调用栈会执行到`fixupProtocol`方法，下面看看这个方法干了啥。

{% highlight cpp %}
static void
fixupProtocolMethodList(protocol_t *proto, method_list_t *mlist,  
                        bool required, bool instance) {
    if (!mlist) return;
    if (mlist->isFixedUp()) return;

    const char **extTypes = proto->extendedMethodTypes();
    fixupMethodList(mlist, true/*always copy for simplicity*/,
                    !extTypes/*sort if no extended method types*/);
    if (extTypes) {
        uint32_t count = mlist->count;
        uint32_t prefix;
        uint32_t junk;
        getExtendedTypesIndexesForMethod(proto, &mlist->get(0), 
                                         required, instance, prefix, junk);
        for (uint32_t i = 0; i < count; i++) {
            for (uint32_t j = i+1; j < count; j++) {
                method_t& mi = mlist->get(i);
                method_t& mj = mlist->get(j);
                if (mi.name > mj.name) {
                    std::swap(mi, mj);
                    std::swap(extTypes[prefix+i], extTypes[prefix+j]);
                }
            }
        }
    }
}
static void 
fixupProtocol(protocol_t *proto) {
    if (proto->protocols) {
        for (uintptr_t i = 0; i < proto->protocols->count; i++) {
            protocol_t *sub = remapProtocol(proto->protocols->list[i]);
            if (!sub->isFixedUp()) fixupProtocol(sub);
        }
    }

    fixupProtocolMethodList(proto, proto->instanceMethods, YES, YES);
    fixupProtocolMethodList(proto, proto->classMethods, YES, NO);
    fixupProtocolMethodList(proto, proto->optionalInstanceMethods, NO, YES);
    fixupProtocolMethodList(proto, proto->optionalClassMethods, NO, NO);

    proto->setFixedUp();
}
{% endhighlight %}

可以看到`fixupProtocol`将当前协议及其遵守的其他协议中的四种方法列表进行fixup，也就是调用 `fixupProtocolMethodList` 方法。
`fixupProtocolMethodList` 首先会调用 `fixupMethodList`，该方法笔者在[类的加载过程](http://www.longjianjiang.com/runtime-source-code-class-load/)有叙述。因为上一步在 `fixupMethodList` 中如果协议的`_extendedMethodTypes` 有值则没有根据方法SEL地址进行排序，所以下面则是对协议中方法列表，根据SEL地址对方法列表进行排序，同时也对协议中`_extendedMethodTypes`方法类型字符串数组排序。

## Protocol Extension

我们都知道Swift可以给协议加extension，而OC中的协议则没有提供添加默认实现的方式，[ProtocolKit](https://github.com/forkingdog/ProtocolKit) 则提供使用Runtime提供这一功能。

使用如下所示:

{% highlight objective_c %}
@protocol HeightProtocol <NSObject>
@required
- (CGFloat)height;
@end

@defs(RemarkProtocol)
- (NSUInteger)score {
    return 100;
}
@end
{% endhighlight %}

defs一个宏，使用defs是因为是一个保留关键字，所以会像`@interface`一样高亮的颜色。

{% highlight cpp %}
#define defs _pk_extension

#define _pk_extension($protocol) _pk_extension_imp($protocol, _pk_get_container_class($protocol))

#define _pk_extension_imp($protocol, $container_class) \
    protocol $protocol; \
    @interface $container_class : NSObject <$protocol> @end \
    @implementation $container_class \
    + (void)load { \
        _pk_extension_load(@protocol($protocol), $container_class.class); \
    } \

#define _pk_get_container_class($protocol) _pk_get_container_class_imp($protocol, __COUNTER__)
#define _pk_get_container_class_imp($protocol, $counter) _pk_get_container_class_imp_concat(__PKContainer_, $protocol, $counter)
#define _pk_get_container_class_imp_concat($a, $b, $c) $a ## $b ## _ ## $c
{% endhighlight %}

> __COUNTER__自身计数器，用于记录以前编译过程中出现的__COUNTER__的次数，从0开始计数。

而上述关于defs的代码展开后则如下所示:

{% highlight objective_c %}
@protocol RemarkProtocol;
@interface __PKContainer_RemarkProtocol_0 : NSObject<RemarkProtocol> @end

@implementation __PKContainer_RemarkProtocol_0
+ (void)load { 
    _pk_extension_load(@protocol(RemarkProtocol), __PKContainer_RemarkProtocol_0.class); 
} 
- (NSUInteger)score {
    return 100;
}
@end
{% endhighlight %}

可以看到defs宏动态创建了一个类，类名三部分组成，__PKContainer_ 前缀，协议名，计数器，这个类中包含了协议中默认的实现方法。

内部又是如何实现遵守协议类增加一个默认实现方法的呢？分为以下几步:

- 通过 `load` 方法，将协议的默认实现方法存储到内部一个 `PKExtendedProtocol` 结构体数组 `allExtendedProtocols` 中；
- 使用 `__attribute__` 编译器指令，使内部 `_pk_extension_inject_entry` 方法在 main 函数执行之前执行；
- `_pk_extension_inject_entry` 内部则通过 `objc_copyClassList` 获取所有类列表 `allClasses`，两层for循环遍历 `allExtendedProtocols` 和 `allClasses`，判断类是否遵守协议，使用 `class_addMethod` 为类动态添加协议方法的默认实现；

笔者这里就不贴代码了，具体可以去看源代码。

## 最后

本文介绍了类的protocol的结构，有了之前类加载的基础，protocol相关的内容显得比较简单了。

## References

[https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_73/rzarg/name_mangling.htm](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_73/rzarg/name_mangling.htm)

[https://stackoverflow.com/questions/2053029/how-exactly-does-attribute-constructor-work](https://stackoverflow.com/questions/2053029/how-exactly-does-attribute-constructor-work)
