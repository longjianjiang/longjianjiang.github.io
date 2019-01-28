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
};
{% endhighlight %}