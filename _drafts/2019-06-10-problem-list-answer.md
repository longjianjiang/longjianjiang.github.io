
# 问题

## 21

wait以后实际mutex是unlock的状态，所以生产者进行生产的时候也需要获取mutex，也就是`enqueue`操作之间要进行加锁，之后进行`broadcast`操作。

进行`broadcast`操作有两种不同的时机，一种是unlock之后，一种是unlock之前。

unlock之后的优点在于，生产者可以立即进行下一次的生产操作(新一次的获取mutex)，但是由于unlock后如果mutex会被其他线程获取，则会导致消费者不能及时的消费。

相对的，unlock之前就不存在前面的问题，因为`broadcast`操作唤醒的线程可以优先获取到mutex，但是一旦被唤醒后线程首先会尝试获取mutex，此时并不会成功，又处于阻塞状态，所以浪费了一些时间，直到unlock后被唤醒的线程才能获取mutex进行消费。

## 20

## 12

如果是HTTP请求建立的TCP连接，服务器短时间不会知道离线。TCP默认的keep-alive机制检测不及时，默认TCP连接7200秒没有发送数据会发送Keep-Alive包，发送一个Keep-Alive包后默认75秒对方没有回应，继续发送下一个，连续发送默认9次Keep-Alive后，没有回应则认为连接失效。对应的三个配置参数为`TCP_KEEPIDLE`, `TCP_KEEPINTVL`, `TCP_KEEPCNT`。
而且如果客户端使用了socks代理，socks代理只会转发具体数据的TCP包，TCP协议实现细节包不会转发，所以此时默认的Keep-Alive机制就失效了。

如果是游戏类建立的TCP连接，因为间隔定时的发送心跳包给服务器，所以服务器可以短时间知道客户端离线。

# References

[https://juejin.im/post/5b5abc145188251b2621f4a4](https://juejin.im/post/5b5abc145188251b2621f4a4)
