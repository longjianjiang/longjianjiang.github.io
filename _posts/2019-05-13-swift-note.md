---
layout: post
title:  "Swift 笔记"
date:   2019-05-13
excerpt:  "本文是笔者记录的一些从开源代码中看到的一些swift语言特性"
tag:
- Swift
comments: true
---

本文笔者记录下自己从开源代码中看到的一些自己没有见过的Swift语言特性，这样更容易去应用到自己写的代码中。

# Trailing Closures

先上一段代码：

{% highlight swift %}
public func dataTask(_: PMKNamespacer, with convertible: URLRequestConvertible) -> Promise<(data: Data, response: URLResponse)> {
    return Promise { dataTask(with: convertible.pmkRequest, completionHandler: adapter($0)).resume() }
}
{% endhighlight %}

笔者刚开始看到上面代码中返回Promise这个还没看明白，初始化怎么直接{},后来想到了这个所谓的TC。

这个特性用的还是比较多的，不过知道了也就习以为常了。

# @autoclosure

先上一段代码：

{% highlight swift %}
func logIfTrue(@autoclosure predicate: () -> Bool) {
    if predicate() {
        print("True")
    }
}

logIfTrue(5 > 7)
{% endhighlight %}

可以看到加了`@autoclosure`后，对于这种closure，我们直接传了一个表达式，编译器会帮我们转成closure的形式。

[ref](https://swifter.tips/autoclosure/)

# enum 

## Associated Values

swift 中的枚举的一个好用的特性就在于可以添加所谓的AV，很多开源库都使用了这个特性。

这个特性允许给枚举的case增加一些需要的值来完成特定的需求，比如一个常见的`Result`枚举：

{% highlight swift %}
enum Result<T> {
	case success(T)
	case failure(Error)
}
{% endhighlight %}

## RawValue 

enum RawValue 和 Associated Values 只能二选一。

因为RawValue是一个`RawRepresentable`协议，如果有RawValue的枚举带了AC，此时`init?(rawValue: Self.RawValue)`初始化的时候，是不确定AC的值的，也就导致了枚举的不唯一。

[ref](https://medium.com/@PhiJay/why-swift-enums-with-associated-values-cannot-have-a-raw-value-21e41d5ec11)

## match

先上一段代码：

{% highlight swift %}
enum Sealant<R> {
    case pending(Handlers<R>)
    case resolved(R)
}

guard case .pending(let _handlers) = self.sealant else {
	return  // already fulfilled!
}
{% endhighlight %}

笔者开始看到guard里加case不知道是干嘛的，后来查了下，其实就是类似`if let`来去除optional这种效果，这里尝试将枚举中的case转为一个特定的case。

不仅仅可以使用上面的`guard case`，还可以使用`if case`和`for case`，具体可以参考[这里](http://alisoftware.github.io/swift/pattern-matching/2016/05/16/pattern-matching-4/)。

## enum codable

枚举实现codable，参考如下：

[ref](https://blog.untitledkingdom.com/codable-enums-in-swift-3ab3dacf30ce)

## enum as

今天笔者看到将字典一个value进行 `as?` 成一个枚举类型，开始卡住as的逻辑是什么，后来想了下其实是设置的时候是enum，因为是Any所以取的时候判断一下类型而已。

# Protocol

## where

在定义协议的时候，如果需要制定遵守协议必须是某个类，可以使用where进行约束，如下：

```swift
protocol ViewProtocol where Self: UIView {}
```

---

在范型中，添加where子句来检查参数类型是否符合条件。

之前说到的枚举中使用`for case`也可以使用where子句来添加限定条件的。

---

for 循环中使用if，swiftlint 会推荐使用where，例如下面例子：
{% highlight swift%}
for item in list where item.isPlaying {
	print("do sth")
}
{% endhighlight %}

## Self

协议中使用到的类型希望就是实现这个协议自身的话，可以使用`Self`进行代替。

```swift
protocol Copyable {
    func copy() -> Self
}
```

实现这个协议的类一是使用final修饰，保证不会有子类继承，如下所示：

```swift
final class Dog: Copyable {
    var name = ""
    func copy() -> Dog {
        let res = type(of: self).init()
        res.name = name
        return res
    }
}
```

二是使用`typeof()`,同时指定一个`required`的init方法，来保证子类也可以响应init方法，如下所示：

```swift
class Person: Copyable {
    var name = ""

    func copy() -> Self {
        let res = type(of: self).init(name: name)
        return res
    }

    required init(name: String) {
        self.name = name
    }
}
```

[ref](https://swifter.tips/use-self/)

# fatalError

先看一个例子：

{% highlight swift %}
class Box<T> {
    func inspect() -> Sealant<T> { fatalError() }
    func inspect(_: (Sealant<T>) -> Void) { fatalError() }
    func seal(_: T) {}
}
{% endhighlight %}

定义了一个Box类，其中两个`inspect`方法调用了`fatalError`，其实可以猜到是父类给出的方法不希望被调用，需要子类自己去实现。

`fatalError`调用后程序中止，给出设定的提示信息。

# subscript(_:default:)

```
subscript(key: Key, default defaultValue: @autoclosure () -> Value) -> Value { get set }
```

Swift Dictionary一个下标运算符，根据名字也可以猜出是干嘛的，当某个key不存在字典中时，使用这个默认值。

[ref](https://developer.apple.com/documentation/swift/dictionary/2894528-subscript)

# IndexSet

一个存储另一个集合中元素idx的set，遍历的时候可以使用`rangeView`方法，根据idx进行生成对应的range。

```
let idxSet: IndexSet = [1, 2, 5, 7]

for range in idxSet.rangeView {
	print(range)
}
```

# Access Control

因为swift没有头文件，所以访问控制需要我们使用特定的标识符来表明代码的访问权限。

一般的业务开始，并不需要关心访问控制，用默认的internal即可，但是模块化的开发中，不可避免的需要写一些framework，这个时候就需要使用到这些标识符。

- public

- open

- internal

- fileprivate

- private

其实看名字能知道一二，下面记录下自己认为需要注意的点。

1. open，public的区别？

区别主要在于类是否能被其他module进行继承从而重载方法。

2. fileprivate 干嘛的？

当同文件中的其他类想要使用某个类的private属性，这个时候可以使用fileprivate。

# Codable

[ref](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types)

# URLComponents & URLQueryItem

在写构建URL的时候，当后面存储参数的时候，写的时候还是比较容易出错的，所以Swift里面提供了两个对象来处理这个事情。

URLComponents 负责构建URL（scheme，host，path），URLQueryItem 用来负责构建参数，这样我们只需要写对应的值，拼接的工作这两个类进行完成。

{% highlight swift%}
let searchTerm = "obi wan kenobi"
let format = "wookiee"

var urlComponents = URLComponents()
urlComponents.scheme = "https"
urlComponents.host = "swapi.co"
urlComponents.path = "/api/people"
urlComponents.queryItems = [
   URLQueryItem(name: "search", value: searchTerm),
   URLQueryItem(name: "format", value: format)
]

print(urlComponents.url?.absoluteString)
// https://swapi.co/api/people?search=obi%20wan%20kenobi&format=wookie
{% endhighlight %}

[ref](https://medium.com/swift2go/building-safe-url-in-swift-using-urlcomponents-and-urlqueryitem-alfian-losari-510a7b1f3c7e)

# init

swift的构造方法分为两种，主要的和便利的。

有规则如下：

1> 主要的构造方法在初始化好自身的属性后，需要调用直接父类的主要构造方法；
2> 便利的构造方法一定需要调用本类的其他构造方法，同时nonetheless最终一定需要调用到本类的主要构造方法；

swift 的初始化分为两个阶段：

第一个阶段是构造方法沿着继承链一直到基类执行完毕，此时对象的所有属性初始化完毕；
第二个阶段是可以访问self，或者调用方法，或者修改类中的属性；

默认子类不会继承父类的构造方法，这样的目的在于防止子类初始化时某些属性没有初始化。当子类的属性有默认值的时候，这个时候有规则如下：

1> 如果子类没有主要的构造方法，此时会继承父类所有的主要构造方法；
2> 如果子类实现了父类所有的主要构造方法，可以是直接继承过来的，也可以是重写的，那么此时子类会继承父类所有的便利构造方法；

`init?` 表示由于某个参数不合适，导致初始化失败，此时返回nil；
`init!` 当确保调用`init?`一定可以返回，可以将init方法加!，来去除返回类型是optional；

[ref](https://docs.swift.org/swift-book/LanguageGuide/Initialization.html)
[ref](https://swifter.tips/init-keywords/)

---

OC中的init和dealloc应该避免使用setter方法去赋值，应该去使用实例变量。

因为setter会带来一些副作用，比如setter会触发KVO通知，也有可能setter方法被重写，在构造和析构的时候，可能会造成一些错误。

{% highlight objc%}
@interface Student : NSObject

@property (nonatomic, copy) NSString *name;

@end


@implementation Student

- (void)dealloc {
    self.name = nil;
}

- (void)setName:(NSString *)name {
    _name = name;

    NSString *tmp = [NSString stringWithString:name];
    NSLog(@"set name is %@", tmp);
}
@end
{% endhighlight %}

dealloc中使用setter触发了自定义的setter，stringWithString方法接受的name为nil，这个方法不允许参数为nil，触发了异常。

[ref](https://stackoverflow.com/questions/8056188/should-i-refer-to-self-property-in-the-init-method-with-arc)

# LazySequence

所谓lazySequence就是延迟计算操作（filter，map）到真正需要使用的时机，而不是立马去执行操作，类似懒加载的用途；

```swift
let arr = [1, 2, 4]
let lazyArr = arr.lazy // LazySequence
let lazyMapArr = lazyArr.map { $0 * 2 } // LazyMapSequence
```

如上代码，`.lazy`会构建一个LazySequence，此时map/filter等操作兼容了LazySequence，返回的结果是LazyMapSequence，这个结构体保存了原数组和transform block，来达到用到才去执行变换操作。

# 断言

```swift
// precondition 可以保证必须满足条件，才会继续往下执行；否则直接终止app（debug & release）
precondition(word.count == 4)
// preconditionFailure 直接终止app（debug & release）
preconditionFailure()

// 断言，debug下执行
assert(condiiton, msg)
// 断言失败，debug下执行
assertionFailure(msg)
```

# guard

guard 算是和if类似的一种选择分支语句。我个人使用guard有两种情况，一种是需要进行对可选类型进行解包，如果失败直接return。

第二种是如果有些条件是符合guard的语义，使用guard进行判断，不符合提前返回。所以这种情况下，不是所有语义都适合guard，有些其实更适合if。

1> 语义：需要保证age大于16，否则不合法。

```swift
guard age > 16 else { return }
```

2> 语义：age小于17，不合法。

```swift
if age < 17 { return }
```

# Existential

TODO

[ref](https://www.dazhuanlan.com/2019/10/16/5da5ff862ce5b/?__cf_chl_jschl_tk__=ae57189f0c7064d19f825a4a3a09e58fc4db9997-1599528725-0-ASF9fqhEjTEHOVigC1B2vll6fe0JBMbOM17I38GqbXuhoJLm5k-xTZnzbRVk8Ppam0BucNSX4-kCxpPlI5-RGqm0L7cYEW3jeT2PLPIUScHyMwIvb_qFII5KrBBAV_HsY57EK-RZIILgss2oYq7qhudNqna7FpWT4W7rGlDx7wY1eB0gQkwOU7MIy14SRlamHcPh4p1I4e8pDNPzDUfeJLqKpuFVccNUI5-QStH3RKBEu3JxrEMxnChCV40iB3zSlRc6W1cof89FFy0HpvZlsZ8xnGBBLilT1ufrWkJVyVDB7NINtW_dPaQcMsBURjyz3g)

[ref](https://tech.meituan.com/2018/11/01/swift-compile-performance-optimization.html)

# function builder (result builder)

先以一个实际的例子来说明这个功能有什么用。

拿构造富文本来说，构造富文本其实有两种思路，第一种思路是我已经知道了整段文本，然后我需要去为每一个子串去设置对应的属性，这样做就会写很多range相关的代码，不可读还容易出错。
所以另一种构造的方案就是，我先构造好每个子串，然后进行拼接，得到整段文本。

function builder 其实可以来完成这件事，其实要做到这事其实很简单，暴露一个构造方法，这个方法接受可变的富文本，然后将这些可变的进行拼接即可，下面就是一个参考实现：


```swift
@resultBuilder
class NSAttributedStringBuilder {
    static func buildBlock(_ components: NSAttributedString...) -> NSAttributedString {
        let res = NSMutableAttributedString(string: "")

        return components.reduce(into: res) { res, str in
            res.append(str)
        }
    }
}

extension NSAttributedString {
    public class func composing(@NSAttributedStringBuilder _ parts: () -> NSAttributedString) -> NSAttributedString {
        return parts()
    }
}
```

@resultBuilder 修饰一个对象，实现一个buildBlock方法，这个方法实现append操作。

总结来说就是，result builder提供给我们一种方式，将一组数据进行组合成一个合成的数据。

SwiftUI 里面的VStack接收的参数其实就是一个result builder修饰的闭包，会将闭包内的各种子view进行打包放到TupleView中。

[ref1](https://www.swiftbysundell.com/articles/deep-dive-into-swift-function-builders/#conditionals)
[ref2](https://www.vadimbulavin.com/swift-function-builders-swiftui-view-builder/)

# References 

[https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html](https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html)

