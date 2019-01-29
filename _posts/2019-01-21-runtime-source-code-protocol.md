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
@required
- (NSUInteger)score;
@end

@protocol HeightProtocol <NSObject>
@required
- (CGFloat)height;
@end

/*****************************************************************/

@interface Person : NSObject<GenderProtocol, HeightProtocol> {
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

- (NSUInteger)score {
    return 100;
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

可以看到 `protocol_t` 也继承自 `objc_object`，所以协议也是一个对象。下面就是协议中包含的属性，很多我们看了也能知道是我们平时写在协议中的哪部分，还有一些则是陌生的，下面笔者来介绍下协议中的属性:

- instanceMethods, classMethods: 协议中的对象方法和类方法列表
- optionalInstanceMethods, optionalClassMethods: 协议中可选的对象方法和类方法列表
- instanceProperties, _classProperties: 协议中对象property和类property(class关键字)
- size: protocol_t 类型的大小
- flags: 32位标记位，最后2位用来存储是否 `fixedup`
- protocols: 继承的其他协议
- _extendedMethodTypes:
- mangledName, _demangledName: 这两个属性关于命名重整。

> C++有所谓命名重整(name mangling)，主要是为了支持函数重载，编译时会根据函数的参数列表的类型生成一个独一无二的名字，防止同名函数的出现，具体可以参考[name mangling](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_73/rzarg/name_mangling.htm)

![runtime_source_code_protocol_1]({{site.url}}/assets/images/blog/runtime_source_code_protocol_1.png)
![runtime_source_code_protocol_2]({{site.url}}/assets/images/blog/runtime_source_code_protocol_2.png)

根据上面两张图可以看到协议中的方法在编译期间就已经加入了`ro`的方法列表中，`ro`的协议列表也存储着当前类遵守的协议。

> 我们可以看到`ro`的协议列表并没有包含父类遵守的协议，因为方法寻找会按继承链去寻找，父类的`ro`的方法列表会添加协议中的方法，所以就没有必要在子类中显示父类的协议了。

![runtime_source_code_protocol_3]({{site.url}}/assets/images/blog/runtime_source_code_protocol_3.png)

还有一个有趣的地方是，一旦某个类遵守了协议后，`ro`的属性列表就会多出四个，这四个就是 `NSObject` 协议中定义的四个属性。

## References

[https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_73/rzarg/name_mangling.htm](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_73/rzarg/name_mangling.htm)