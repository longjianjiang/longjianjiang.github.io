# 多线程具体问题

## 请求依赖

AB两个请求，B请求需要用到A请求返回值。

所以一定是有一个监听的操作，当A请求回来后，再发起B请求。

1> 朴素方式，A请求的callback里面发起B；或者用RxSwift将请求包装成一个信号，然后使用flatMap进行包装一下，其实本质和朴素方式一样，只是框架的存在让代码更好看一些。

2> 将请求放到一个串行队列中，那么串行队列会等同步/异步任务完成后，才去执行下一个任务。

3> apiManager的形式，接口层面添加依赖的方法，每个apiManager会有一个依赖的set、自身的一个初始化为0的信号量。当自身请求成功好了后，会signal信号量，而每个发起请求的apiManager则会等候set里所有的都完成之后在请求。

## 统一回调

同时发起若干请求，需要在所有请求完成时进行处理。

1> 使用dispatchGroup，其实是对信号量的一次封装。

2> 设置标记位，互斥的改变标记位，直到所有任务好了，执行回调，类似zip操作。
## 限制最大并发

若干任务同时执行，限制同时只能有两个任务被执行。

1> 使用信号量，首先创建一个串行队列，负责执行判断信号量的异步任务。创建一个并行队列，负责执行异步的IO任务。

[ref](https://github.com/buaa0300/QSDispatchQueue/blob/master/QSDispatchQueue/QSDispatchQueue.m)

2> 创建串行队列池，每次来新的任务，从池中取一个串行队列投放一个任务去执行，利用多个串行队列来达到并发同时控制线程。

[ref](https://github.com/ibireme/YYDispatchQueuePool/)