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

这个方法就是对类进行第一次加载的，下面笔者就一步一步的展示这一过程。


