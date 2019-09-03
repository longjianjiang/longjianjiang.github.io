
# 问题

## 21

wait以后实际mutex是unlock的状态，所以生产者进行生产的时候也需要获取mutex，也就是`enqueue`操作之间要进行加锁，之后进行`broadcast`操作。

进行`broadcast`操作有两种不同的时机，一种是unlock之后，一种是unlock之前。

unlock之后的优点在于，生产者可以立即进行下一次的生产操作(新一次的获取mutex)，但是由于unlock后如果mutex会被其他线程获取，则会导致消费者不能及时的消费。

相对的，unlock之前就不存在前面的问题，因为`broadcast`操作唤醒的线程可以优先获取到mutex，但是一旦被唤醒后线程首先会尝试获取mutex，此时并不会成功，又处于阻塞状态，所以浪费了一些时间，直到unlock后被唤醒的线程才能获取mutex进行消费。

## 20

runtime在运行时会判断当前类的`instanceStart`是否小于父类的`instanceSize`，如果小于说明父类增加了成员，需要进行动态调整子类的内存布局，也就是所谓的`non fragile ivars`。

具体调整逻辑在`moveIvars`方法中，大致逻辑是计算出diff和ivars中的最大的对齐值，将diff按这个值进行对齐，然后用diff更新ivars的offset，最后更新ro的`instanceStart`和`instanceSize`。

## 19

`[super dealloc]` 其实是对obj发送`dealloc`消息，ARC中是根据引用计数来自动释放内存的，所以不允许手动发送释放内存的消息。

## 18

```
SideTables 存在的意义？
```

## 17

主要是让编译器对分支转移进行优化，减少指令的跳转。

## 16

是的，这样让线程一直处于工作的状态，当有任务来临的时候进行处理，没有任务则处于休眠状态。

## 15

double free一个指针出错在于，内存分配的机制中会区分该块内存是否是已分配的，所以第二次free自然会出错。

访问freed指针出错的原因在于有可能这部分内存已经被重新分配，所以以原来对象的方式去使用会出错；相对应的这部分内存如果没有重新分配使用，那么这个时候对这块内存进行读写是不会出错的，因为向内核申请的内存页依然存在，只是此时内存是处于未分配状态，所以一般free后可以将指针指向NULL。

## 14

函数调用通过栈来实现，每次函数调用保存上一次函数调用的栈基址，也就是`push %ebp`操作。

获取当前函数的栈基址，找到该地址在栈中的值，也就是caller的栈基址，如此重复，就可以获得函数的调用栈。

因为需要操作寄存器，所以实现应该在内核态中。

## 13 

不知道这里的`原始数据`指的是什么？

ssl握手阶段是明文传输，第三个随机数使用DH算法来增加安全性，随后使用三个随机数来生成一个key进行后续对称加密的传输。

## 12

如果是HTTP请求建立的TCP连接，服务器短时间不会知道离线。TCP默认的keep-alive机制检测不及时，默认TCP连接7200秒没有发送数据会发送Keep-Alive包，发送一个Keep-Alive包后默认75秒对方没有回应，继续发送下一个，连续发送默认9次Keep-Alive后，没有回应则认为连接失效。对应的三个配置参数为`TCP_KEEPIDLE`, `TCP_KEEPINTVL`, `TCP_KEEPCNT`。
而且如果客户端使用了socks代理，socks代理只会转发具体数据的TCP包，TCP协议实现细节包不会转发，所以此时默认的Keep-Alive机制就失效了。

如果是游戏类建立的TCP连接，因为间隔定时的发送心跳包给服务器，所以服务器可以短时间知道客户端离线。

## 11

Swift String由Character组成，每个Character在unicode编码方式下占的位数是不一样的，所以如果使用Int作为下标来取字符就会出错，所以String添加了Index结构体来表示每个字符的下标，下标运算符只接受Index类型。

String提供了`startIndex`表示第一个字符的下标，`endIndex`表示最后一个字符的后一个下标，用来作为边界判断。

## 10

NSDictionary: 用来存储key&value形式的数据，使用哈希表来实现，所以内部会强制对key进行copy，这样对应value的位置就不会变。
要求key实现`NSCopying`，所以当key是自定义对象的时候，强制copy会影响性能，所以这个时候需要提供一种不拷贝的选择。可以使用`NSMapTable`。

NSHashTable: 类似NSSet，存储不重复的元素，NSSet内部会对持有元素，也就是对元素进行一次retain操作。当需要存储弱引用的时候，就需要使用到NSHashTable，该类提供了一个`weakObjectsHashTable`快捷方法来专门存储弱引用。

NSHashTable相比NSSet，提供了自定义的内存管理选项，不仅仅是默认的`strong`的方式，同时也提供了不copy的选项，最后提供了hash和equality的方式。这三个方面，Apple提供了定义好的`NSPointerFunctionsOptions`，我们按需组合即可。

## 9

1. 使用静态全局变量，因为存放在静态区域，此时生成的block结构体中不会添加成员，直接使用外部的全局变量，所以block内部进行修改没有任何问题。

2. 使用静态局部变量，这个时候生成的block结构体中会存储一个该变量的指针，通过指针可以修改静态局部变量。

使用上述两种方式局限在于block销毁后，被捕获的变量不能及时的被释放。

## 8

isa是一个联合，其中一个成员是cls，也就是对象指向的类，因为64位地址位数较多，这个时候isa成员中有一个结构体，结构体中44位存储了类的信息，还存储了对象的一些额外的信息。这个时候对象的指针叫做所谓的`nonpointer`。

isa 是OC对象的一个标志，有了isa，也就有了消息机制。对象的isa指向类，类的isa指向原类，这样调用实例方法和类方法可以走同一套流程。

## 7

id 是`struct objc_object *`的别名。

self 是指向自身的一个指针，用self进行方法调用，也就是从当前对象isa的Class继承链中根据selector寻找方法实现。

super是一个编译器修饰符，使用super调用，会从父类Class开始进行根据selector进行寻找方法实现。第一个receiver参数还是self。

## 6

用STL的容器存储OC的对象，相当于存储指向结构体的指针。

自己用`vector`做了一个测试，发现vector会将添加的对象进行一次retain，当栈上的vector销毁时，会对vector内的对象进行一次release。

## 5

placement new是将对象的创建内存分配和初始化分开，用提前分配好的内存去给某个对象进行初始化。

objc中实现这个功能，等于就是把`alloc`中calloc分配内存和初始化isa的工作给拆分开来，需要在`objc_object`中增加一个placement new的方法，外界根据`class_getInstanceSize`获取类的大小，提供已分配的内存，新增方法中直接初始化isa即可。

## 4

cpp中调用虚方法是通过虚函数表地址去虚函数表中找到对应的方法实现，完成调用。每个类会维护一个虚函数表，如果没有子类没有重写父类的虚方法，表中指向相同的函数实现地址。
类中非虚方法则是编译期间就固定了函数实现地址。

objc中的方法调用由runtime进行完成，实现通过objc_msgSend，大致分为以下几步：
1. 根据object获取Class；
2. 首先尝试从类的缓存中根据selector找方法实现；
3. 如果缓存中找不到，则根据selector按类继承链寻找方法实现；
4. 如果类继承链中找不到，到消息决议阶段；
5. 消息决议失败，到消息转发阶段；
6. 消息转发失败，走到doesNotRecognizedSelector；

objc的运行时的消息转发动态性更强，类中所有的成员方法都是“虚的”。

## 3

fishhook是一个可以hookC语言系统函数的库。原理是通过`_dyld_register_func_for_add_image`注册一个回调，没当dyld加载新的动态连接库的时候以及已经加载进内存的动态连接库都会执行一次回调。这个时候fishhook会进行查找symbol，进行替换函数指针，达到hook的效果。

所以非动态连接库的符号自然是不能通过fishhook进行修改的。

## 2

## 1

1> 

UIView继承自UIResponder，可以处理用户点击事件。

CALayer用来管理可视内容。UIView里有一个layer属性，其实UIView是CALayer的一个代理，当CALayer需要display的时候，这个时候会调用代理也就是UIView的`drawRect:`方法进行内容的绘制。
绘制完成后会将layer的contents进行渲染，这一步由Core Animation完成。

因为渲染操作并不是在UIKit中实现，所以这部分渲染的逻辑可以复用，实际上UIKit与AppKit就是共用了同一套渲染逻辑。

2>

前面说到`drawRect:`用来进行内容的绘制。不知道这里所说的性能具体是哪方面，屏幕的刷新率是60fps，也就是说16.7ms会进行一次刷新，而内容展示由绘制和渲染两步组成，分别由CPU和GPU来完成。可以通过time profiler进行实际的观察。

3>

最本质的区别应该是UI Dynamic引入来动力模型，类似2D物理引擎。实现类似重力，碰撞效果更加逼真。

# References

[https://juejin.im/post/5b5abc145188251b2621f4a4](https://juejin.im/post/5b5abc145188251b2621f4a4)
