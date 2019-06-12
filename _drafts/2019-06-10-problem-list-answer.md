
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

## 12

如果是HTTP请求建立的TCP连接，服务器短时间不会知道离线。TCP默认的keep-alive机制检测不及时，默认TCP连接7200秒没有发送数据会发送Keep-Alive包，发送一个Keep-Alive包后默认75秒对方没有回应，继续发送下一个，连续发送默认9次Keep-Alive后，没有回应则认为连接失效。对应的三个配置参数为`TCP_KEEPIDLE`, `TCP_KEEPINTVL`, `TCP_KEEPCNT`。
而且如果客户端使用了socks代理，socks代理只会转发具体数据的TCP包，TCP协议实现细节包不会转发，所以此时默认的Keep-Alive机制就失效了。

如果是游戏类建立的TCP连接，因为间隔定时的发送心跳包给服务器，所以服务器可以短时间知道客户端离线。

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

# References

[https://juejin.im/post/5b5abc145188251b2621f4a4](https://juejin.im/post/5b5abc145188251b2621f4a4)
