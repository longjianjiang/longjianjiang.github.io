---
layout: post
title:  "【Tips】拒绝循环引用"
date:   2017-03-01
excerpt: "最近项目中检查是否有内存泄露，发现大部分的情况都是由于block的循环引用而导致的内存泄露，所以借此机会来记录下常见的循环引用的例子，如果不全，还请各位在评论区多多补充。"
tag:
- iOS
comments: true
---


### 前言
最近项目中检查是否有内存泄露，发现大部分的情况都是由于block的循环引用而导致的内存泄露，所以借此机会来记录下常见的循环引用的例子，如果不全，还请各位在评论区多多补充。(哈哈，抛砖引玉☺️)


#####  1)

{% highlight objective_c %}
@interface LJTestViewController ()

@property (nonatomic,copy) myBlock block;

@end

@implementation LJTestViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.block = ^(NSString *name){
        NSLog(@"%@",self);
    };
    
}

@end
{% endhighlight %}

这种情况很明显了(PS:Xcode会有提示😶)，因为在控制器持有的block中使用了`self`,所以block内部持有控制器本身，由此导致了循环引用，解决方法也很简单,设置一个弱引用即可。同时此时block存在的时候，`self`也必定存在，所以不需要判断是否为`nil`.


#####  2)

{% highlight objective_c %}
#import "LJTestViewController.h"
#import "Person.h"
@interface LJTestViewController ()

@property (nonatomic,copy) myBlock block;

@property (nonatomic,strong) Person *p;
@end

@implementation LJTestViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.block = ^(NSString *name){
        NSLog(@"%@",_p);
    };
    
}


@end
{% endhighlight %}

这种情况Xcode同样会给出提示的，其实和上一种是一样的情况，虽然看上去block内部没有持有`self`，但其实编译器读的时候把`_p` 转为`self->p`，这样其实就相当于持有了`self`。解决方案可以在block外部用一个局部变量指向`_p`，block内部使用声明的局部变量即可。


#####  3)

{% highlight objective_c %}
#import "LJTestViewController.h"
#import "Person.h"
@interface LJTestViewController ()
@property (nonatomic,copy) myBlock block;


@property (nonatomic,strong) Person *p;
@end

@implementation LJTestViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    
    Person *p = [Person new];
    p.block = ^(NSString *name) {
        self.title = name;
    };
    
    self.p = p;

}

@end
{% endhighlight %}

这种情况就有点隐蔽了，因为此时Xcode并没有提示了；其实他和前面两种情况也是类似的，只不过经过中间一次赋值，但达到的效果还是一样的。一般的我们开发中的循环引用我想大概基本上都是由于这种情况造成的。但解决方案还是和之前一样。所以当某个对象持有block的时候或者某个对象的对象持有block的时候，此时如果该block内部有使用了最前面的那个对象那么此时就会出现循环引用:

![屏幕快照 2017-03-01 下午3.27.11.png]({{site.url}}/assets/images/blog/retain_cycle.png)

解决方案如下：

{% highlight objective_c %}
    __weak typeof(self) weakSelf = self;
    self.block = ^(NSString *name) {
        __strong typeof(weakSelf) strongSelf = weakSelf;  // 防止self为nil
        if (strongSelf) { // 此时strongSelf也有可能为nil
            NSLog(@"%@",strongSelf);
        }
    };
{% endhighlight %}


#####  4)

{% highlight objective_c %}
#import "LJTestViewController.h"

@interface LJTestViewController ()

@property (nonatomic,strong) id<NSObject> observer;
@end

@implementation LJTestViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    
    self.observer = [[NSNotificationCenter defaultCenter] addObserverForName:@"someNotification" object:nil queue:nil usingBlock:^(NSNotification * _Nonnull note) {
        NSLog(@"%@",self);
    }];


}

- (void)dealloc {
    NSLog(@"dealloc");
    [[NSNotificationCenter defaultCenter] removeObserver:self.observer];
}

@end
{% endhighlight %}

这个是最近刚发现的一个循环引用，当用到带block的通知的时候，如果block中使用了`self`,需要用弱指针指向；否则会导致控制器无法销毁。具体原因并不清楚，如果各位有相关的资料或者合理的解释，可以随时联系我。


#####  最后)

{% highlight objective_c %}
[UIView animateWithDuration:5 animations:^{
    // some animation code
    self.view.backgroundColor = [UIColor orangeColor];
}];


[[NSOperationQueue mainQueue] addOperationWithBlock:^{
    self.view.backgroundColor = [UIColor orangeColor];
}];



dispatch_async(dispatch_get_global_queue(0, 0), ^{
    self.view.backgroundColor = [UIColor orangeColor];
});
{% endhighlight %}

类似上面的系统API中的block中引用到`self`的情况并不会造成循环引用，因为此时block没有被`self`所持有，所以在block中使用`self`是不会发生什么问题的。


# Swift中的循环引用 更新于2017-11-12

上面所说的解决循环引用的是在OC中，Swift下相比OC中多了一个关键字`unowned`，作用和`weak`一致，都是设置为弱引用。

我们知道OC中设置为weak的属性，访问时如果指向的实例被释放，此时该属性会自动设置为nil，Swift中同样也是，所以设置为weak的属性，同样也是可选的。
而unowned却不一样，访问时如果指向的实例被释放，此时该属性不会被设置为nil，所以此时就会出错，所以在选择是勇weak还是unowned时，我们需要判断对象的生命周期，来进行选择。

> 上述使用的unowned是安全的，Swift同样提供了一种不安全的，`unowned(unsafe)`, 当去方式被释放的实例时，此时会去实例所在内存地址读取，这是一种不安全的操作，更多的时候我们使用默认的安全类型。

