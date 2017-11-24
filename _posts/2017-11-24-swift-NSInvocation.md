---
layout: post
title:  "【Tips】Swift 下NSInvocation 的 “替代方案” "
date:   2017-11-24
excerpt: "某些场景下Swift中对NSInvocation的替代方案。"
tag:
- Swift
comments: true
---

  > 我们知道OC中的方法调用在运行时都会转化为消息的方式，NSInvocatin就是封装了消息的一个类，使用它可以达到发消息也就是调用方法的效果。

# 开始
因为在ResponderChain的对象交互方式使用strategy模式进行事件处理OC中可以使用NSInvocation， 但是Swift中并不支持NSInvocation，所以笔者查阅资料做了一个类似的方案，相当于使用C语言函数去调用，达到了invoke的效果，代码如下：

```
 func extractMethodFrom(owner: AnyObject, selector: Selector) -> (([String : Any]?) -> Void)? {
        let method: Method?
        if owner is AnyClass {
            method = class_getClassMethod(owner as? AnyClass, selector)
        } else {
            print(type(of: owner))
            method = class_getInstanceMethod(type(of: owner), selector)
        }
        
        if let one = method {
            let implementation = method_getImplementation(one)
            
            typealias Function = @convention(c) (AnyObject, Selector, [String : Any]?) -> Void
            
            let function = unsafeBitCast(implementation, to: Function.self)
            
            return { userinfo in function(owner, selector, userinfo) }
            
        } else {
            return nil
        }
 }
```

这样我们类似维护这样一个事件数组：

> 注意： 这里的selector得是OC中的，不然通过class_getxxx 方法返回会为空,并且方法也必须是标记@objc的；

```
let eventStrategy: [String: String] =  [kCTOrderProfileFooterEventTappedReceivingBtn: "clickFooterBtn:"]
```

然后我们在router中这样调用：

```
 override func routerEventWithName(_ eventName: String, userInfo: [String : Any]?) {
        let methodName = eventStrategy[eventName]
        if let name = methodName {
            let selector = NSSelectorFromString(name)
            if let method = extractMethodFrom(owner: self, selector: selector) {
                method(userInfo)
            }
        }
        
    }

```

OC中router如下：

```
- (void)routerEventWithName:(NSString *)eventName userInfo:(NSDictionary *)userInfo {
    NSInvocation *invocation = self.eventStrategy[eventName];
    [invocation setArgument:&userInfo atIndex:2];
    [invocation invoke];
}
```

一个通过NSInvocation设置参数，一个通过函数设置参数，效果是类似的。


# 最后

觉得这个方式还是挺不错的，所以记录下来。
