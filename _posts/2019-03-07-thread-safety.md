---
layout: post
title:  "多线程学习笔记"
date:   2019-03-07
excerpt:  "本文是笔者学习多线程的笔记"
tag:
- OS
comments: true
---

> 多线程问题不可否认是一个复杂的问题，本文笔者将自己对多线程的学习和理解记录下来。

## 引子

首先我们想一下阻塞队列和并发队列如何实现？其实我们需要解决的问题就是生产者和消费者的问题，我们需要达到的目的就是不同个数的生产者和消费者可以正常的操作物品，生产者不会在存储满时继续生产，消费者也不会在存储为空时继续消费。

上述所说的也就是所谓的多线程同步问题。

阻塞队列我们可以通过`mutex`和`conditoin_variable`实现同步，实现方式是通过阻塞线程。

并发队列则以不阻塞线程的方式达到线程的同步，通过原子操作，也就是CAS（Compare And Swap）。

## 线程

我们知道一个程序其实就是一堆指令的集合存储在硬盘上，当我们运行这个程序的时候，操作系统会将这个程序的指令集合加载到内存，此时这个程序就会产生一个运行的实例，称之为进程。现在CPU基本都是多核，所以为了提高CPU执行的效率，就需要充分利用多核，所以此时出现了线程的概念，线程是操作系统进行任务调度的最小单元，拥有自己独立的栈，存在与进程中。一条线程可以理解为进程中的一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。

C++11中在标准库中增加了线程的相关API，其中创建和管理线程使用的是 `std::thread` 类。

{% highlight cpp %}
void do_something() {
    std::cout << "thread do something\n";
}

int main(int argc, const char * argv[]) {
    std::thread td{do_something};
    td.join();
    return 0;
}
{% endhighlight %}

上述例子中创建了一个线程 td，指定 do_something 作为线程的入口函数，也就是说当这个线程一创建就会执行 do_something。

> 此时我们称 td 为 `worker_thread`, td 在主线程上发起，此时称主线程为 `launch_thread`;

`td.join()` 这个方法是我们指定 td 在销毁之前如何处理线程的结束状态，这里的 `join()` 的效果是，`launch_thread` 会在调用`join()` 后阻塞，直到 `worker_thread` 执行完成后继续执行。

还有另一个选择是 `detach()`，此时 `worker_thread` 创建后立即和 `launch_thread` 分离，此时两条线程各自独立执行。此时需要注意的是，当 `launch_thread` 结束并伴随着资源的销毁，`worker_thread` 可能仍在运行，此时需要保证 `worker_thread` 没有引用到这些资源。

如果没有没有指定 td 在销毁之前如何处理线程的结束状态，线程会在析构函数中调用 `std::terminate` 造成系统崩溃。

也就是对于我们创建的线程，我们必须调用 `join()` 或者 `detach()`，我们可以创建一个类来处理线程的结束状态，如下所示:

{% highlight cpp %}
struct thread_guard {
    std::thread &t;
public:
    explicit thread_guard(std::thread& t_): t(t_) {}
    ~thread_guard() {
        if (t.joinable()) {
            t.join();
        }
    }
    thread_guard(thread_guard const&) = delete;
    thread_guard& operator=(thread_guard const&) = delete;
};

int main(int argc, const char * argv[]) {
    std::thread td{do_something};
    thread_guard g{td};
    do_domething_in_current_thread();
    return 0;
}
{% endhighlight %}

因为 `thread_guard` 对象 g 在 td 之前销毁，同时 g 析构时调用了 `join()`，从而保证了 td的安全释放。

上述方式就是所谓的 RAII(Resource Acquisition Is Initialization)，将某个资源和局部变量的生命周期绑定在一起，因为C++中允许栈对象的创建和析构，保证了我们可以正常的使用资源，不必担心泄露的问题。

进程中的多个线程会共享进程空间中的数据，这个时候如果多个线程对同一个数据进行写入操作，则会出现一些意外的结果，比如下面的例子：

{% highlight cpp %}
int num = 0;
void thread_proc() {
    for (int i = 0; i < 10000; ++i) {
        ++num;
    }
}

int main(int argc, const char * argv[]) {
    std::thread td1(thread_proc), td2(thread_proc);
    td1.join();
    td2.join();
    std::cout << num << std::endl;

    return 0;
}
{% endhighlight %}

上述代码运行结果其实我们希望的是20000，因为多线程的原因，实际运行的结果是不确定的。

因为 `++num` 这个语句翻译成汇编指令，需要三步，首先将num的值读入寄存器，增加寄存器中值，将寄存器中的值写入num，而操作系统在执行的时候，由于中断的存在，可能线程1执行到第二步的时候被中断，此时执行线程2了，所以导致了结果的不确定性。

上述例子证明了，多线程下存在安全问题，所以下面笔者会介绍应对方案。

## 阻塞型同步

### mutex

mutex（mutual exclusion）的基本逻辑是，当某个线程获取到某个资源的控制权时，这个时候其他线程只能等待另一线程释放控制权后才能进行访问。

C++标准库中提供了 `std::mutex` 来操作 mutex，但实际我们并不会直接操作mutex，因为`unlock`操作无法覆盖到所有执行路径，比如当异常发生时。这个时候其他线程永远无法获取到mutex，则会处于永远的阻塞状态。

所以我们依然会使用之前说的 RAII 将其包装起来，C++标准库中也提供了这个包装类 `std::lock_guard`。

### deadlock

什么情况下会发生死锁？比如现在有两个线程需要同时获取两个资源，每个资源有一个mutex，此时线程1获取到了资源1的mutex，开始尝试获取资源2的mutex，但是此时资源2的mutex被线程2所获取，而且线程2也在尝试获取资源1的mutex，此时两个线程互相等待，也就是所谓的死锁。

为了解决死锁，C++标准库中提供了 `std::lock()` 函数用来同时锁住多个mutex。

但是导致死锁并不是只要对mutex的互相等待这一种情况，比如两个线程相互join也会导致死锁。所以避免死锁的关键在于防止两个线程互相等待的发生，下面给出几种有效的方式：

1.避免锁的嵌套，当已经获取到一个锁时，不要尝试在获取第二把锁，如果真的需要则使用 `std::lock`。

2.当获取到某个资源的锁时，避免调用不可控的外界函数，防止对共享数据做一些未定义的行为。

3.按顺序上锁解锁，保证有序性。

### std::unique_lock

之前我们使用`std::lock_guard`，必须等到该实例销毁时，才会unlock对应的mutex，这样会存在一些问题。比如在函数执行完之前需要做一些其他操作，这部分操作的存在破坏了锁的粒度，同时也会降低了并发的效率，导致了其他线程等待时间的增加。所以此时需要一个更加灵活的包装类，C++标准库中提供了 `std::unique_lock`。

`std::unique_lock` 有和mutex一样的 `lock()`, `try_lock`, `unlock` 方法。这样我们可以提前unlock，保持锁的最小粒度。

`std::unique_lock` 的实现其实也不复杂，和`std::lock_guard`一样，内部持有了一个mutex，但是额外增加了一个bool变量标记是否持有mutex，在析构时bool变量为true才执行mutex的 unlock 方法。

### std::shared_mutex & std::shared_lock

当一个操作读操作频率远大于写操作时，此时使用通常的mutex会影响性能，因为多线程下多个线程同时进行读操作是没有问题的，所以在读操作时我们可以不阻塞，当进行写操作时，此时不论读还是写操作都是不安全的，此时阻塞，这样可以提高性能。

C++17标准库为我们提供了这种读写锁 `std::shared_mutex`，额外增加了 `lock_shared`, `try_lock_shared`, `unlock_shared`。所以我们在读操作时使用 `lock_shared` 这样就不会阻塞其他进行读操作的线程，写操作时使用 `lock` 阻塞其他线程保证线程安全。

C++14标准库中提供了 `std::shared_lock`, 和之前的`std::unique_lock`类似，只是`lock`和`unlock`操作被替换为 `lock_shared` 和 `unlock_shared`。

### std::recursive_mutex

上述提到的mutex，如果在一个线程中被连续的lock多次，会产生死锁，而且会出现未定义行为。如果实际情况中真的需要对同一个mutex进行多次lock操作，C++提供了 `std::recursive_mutex`, 所谓的递归锁。递归锁释放时，需要调用相同数量的unlock，才能释放mutex。

### std::once_flag & std::call_once

C++11提供了 `std::once_flag`,  `std::call_once` 来进行单例初始化。如下例子所示：

{% highlight cpp %}
volatile T* pInst = nullptr;
std::once_flag flag_T;                          

void ConstructInstance() {                     
    pInst = new T;
}

T* GetInstance() {
    std::call_once(flag_T, ConstructInstance);  
    return pInst;
}
{% endhighlight %}

### conditoin_variable

条件变量通过消息机制来处理多线程同步问题，可以阻塞一个或多个线程直到收到唤醒通知或者超时从而继续执行。

C++11提供了 `std::conditoin_variable` 必须和 `std::unique_lock` 搭配使用，`std::conditoin_variable_any` 可以搭配任意类型的锁使用。

下面给出一个条件变量的时序图:

![thread_safety_1.png]({{site.url}}/assets/images/blog/thread_safety_1.png)

{% highlight cpp %}
template <class _Predicate>
void
condition_variable::wait(unique_lock<mutex>& __lk, _Predicate __pred)
{
    while (!__pred())
        wait(__lk);
}
{% endhighlight %}

根据上述代码和上图我们可以思考下两个问题：

1.能不能把while换成if？

2.为什么wait方法有一个mutex的参数？

问题1:     
因为存在虚假唤醒的存在，所以需要使用while循环来保证被唤醒后条件是否真的满足。wait在linux中使用 `futex` 的系统调用。当进程被信号中断后，之前的所有阻塞系统调用都会被中止，直接返回。所以此时如果没有while循环去检查条件是否满足，则会出错。

问题2:     
可以看到wait有两个步骤，首先检查条件是否满足，然后决定是否调用wait，可以看到有两步。如果在这两步之间有一个线程将条件改成true，正常情况应该就不需要wait了或者应该被唤醒了，但是实际确是处于一直等待状态。

因为wait有两步的原因，导致了两步之间的空隙可能其他线程会进行操作，所以mutex的参数就是为了来防止这种情况的发生。

这个时候就如上图所示，到调用wait时，首先wait内部会将mutex unlock，进入等待。当被唤醒时进行lock，如此只要其他线程修改条件正确加锁了，那么就不会出现之前的情况。

### semaphores

C++20中引入了 `semaphores`，也就是所谓的信号量，现在还未加入标准库。

所谓信号量其实就是大家公用一个counter，当counter为0时就阻塞。而且存在两个操作，counter+1，counter-1。这样多个线程操作同一个信号量，达到线程同步的效果。

## 非阻塞型同步

### atomic

在阻塞型同步我们说了不同中的mutex，这种类似的锁我们称为悲观锁。可以看到的是这种锁每次都假定会有其他线程会来争夺资源，所以首先会进行lock拿到锁，才进行操作。

对应的就有乐观锁，所谓乐观锁假定不会有其他线程来争夺资源，所以可以直接操作资源。或者运气不佳正好遇到了其他在操作的线程，也没关系，进行重试就好，直到成功。

现在就存在一个问题，没有锁，那如何知道一次操作是否和其他线程冲突呢？这个时候就需要原子操作，也就是在执行这种操作时不会被打断。而原子操作一般通过CAS操作实现。如下所示是C++中提供的CAS方法(执行过程是不可分割，也就是以原子的方式进行)：

{% highlight cpp %}
template <class _Tp>
inline bool
atomic_compare_exchange_weak(atomic<_Tp>* __o, _Tp* __e, _Tp __d) _NOEXCEPT;

template <class _Tp>
inline bool
atomic_compare_exchange_strong(atomic<_Tp>* __o, _Tp* __e, _Tp __d) _NOEXCEPT;
{% endhighlight %}

上述方法会首先会比较对象包含的值和expected指向的值是否相等:         
1.如果相等，则替换对象的值为desired，返回true，这个操作称之为Read-Modify-Write；     
2.如果不想等，则替换expected指向的值为对象的值，返回false；     

上述weak和strong的版本区别在于，weak版本允许spurious failures（伪失败），所谓伪失败指的是对象包含的值和expected指向的值是否相等也可能返回false，这种情况不会进行替换。在一些平台上面，CAS操作是用一个指令序列来实现的，不同与x86上的一个指令。在这些平台上，切换上下文，另外一个线程加载了同一个内存地址，种种情况都会导致一开始的CAS操作失败。称它是假的，此时CAS失败并不是因为存储的值与期望的值不相等，而是时间调度的问题。CAS的strong版本的行为不同，它把这个问题包裹在其中，并防止了这种伪失败的发生。

一般的建议是在循环中使用weak版本的，此时性能会比较高，其他情况则使用strong版本的。

### ABA问题

下面我们来看一个实际的例子，往链表头部插入新节点，代码如下所示:

{% highlight cpp %}
struct list {
    std::atomic<node*> head;
};
 
void append(list* s, node* n) {
    node* head;
    do {
        head = s->head;
        n->next = head;
    } while (!std::atomic_compare_exchange_weak(&(s->head), &head, n));
}
{% endhighlight %}

假设目前链表为 `b->c->nil`，我们使用上面的append方法在b之前插入a节点。线程1在执行CAS之前，线程2先将b移除，插入d，然后再将b插入，此时链表变成了 `b->d->c->nil`，这个时候线程1执行CAS的时候，是无法察觉的，虽然head没变，但是链表的结构已经变了，这种问题就是所谓的ABA问题。这里问题的关键在于，虽然比较的时候b是一样的，但是并不意味着没有发生变化，

### memory order

之前说到的CAS的方法，内部是调用C++中 `std::atomic` 对象的CAS方法，以strong为例，共有4个重载声明:

{% highlight cpp %}
bool compare_exchange_strong(_Tp& __e, _Tp __d,
                                 memory_order __s, memory_order __f) volatile
bool compare_exchange_strong(_Tp& __e, _Tp __d,
                                 memory_order __s, memory_order __f) 
bool compare_exchange_strong(_Tp& __e, _Tp __d,
                                 memory_order __m = memory_order_seq_cst) volatile 
bool compare_exchange_strong(_Tp& __e, _Tp __d,
                                 memory_order __m = memory_order_seq_cst)
{% endhighlight %}

可以看到多了叫 `memory_order` 的参数，这个参数是用来指定内存序。我们实际编写的代码经过编译器的编译后，编译器会做一些优化，从而更加高效的利用CPU，所以可能造成的一个现象就是所谓的乱序执行，即实际运行的顺序并不一定按照书写的顺序。此时如果是单线程没有问题，但是在多线程的环境下就可能出现问题。

```
std::atomic<int> ai = 0;
std::atomic<int> aj = 0;

Thread1:        Thread2:
ai = 100;       while(aj != 200)
                    ;
aj = 200;       std::cout << ai;
```

如上代码所示，如果线程1执行顺序按照书写顺序，那么线程2会输出100；但是实际情况是可能线程1的执行会乱序，也就是 `aj = 200` 会提前执行，所以这个时候线程2的输出就有为0了。而且就算没有乱序执行，CPU 缓存的原因，线程2可能不能立即看到ai被修改的新值，此时线程2仍然有可能输出0。

memory order 就是用来指定原子操作周围的非原子操作的内存访问如何被排序和同步。可以看到默认使用的是 `std::memory_order_seq_cst` ，这是最严格的memory order，对性能一定损耗。下面分别介绍6种不同的内存序。

`std::memory_order_acquire` 使用在load操作，也就是读操作。使用它可以保证load之后的读写操作不允许移动到load前面。

`std::memory_order_release` 使用在store操作，也就是写操作。使用它可以保证store之前的读写操作不允许移动到store后面。

`std::memory_order_consume` 和acquire类似，同样用于load操作。使用它可以保证load之后的和当前load变量有依赖的读写操作不允许移动到load前面。

下面看一个例子:

{% highlight cpp %}
std::atomic<int> ai = ATOMIC_VAR_INIT(5);
int flag = 0;

void thread_1() {
    flag = 1;
    ai.store(1, std::memory_order_release);
}

void thread_2() {
    while (ai.load(std::memory_order_acquire) == 0)
        ;
    assert(flag = 1);
}
{% endhighlight %}

可以看到在线程2，我们断言了flag一定为1，这是为什么呢？当线程1 store的值被线程2 load到，此时线程2可以看到线程store之前对内存所有的读写，所以线程2可以断言flag一定为1。

`std::memory_order_relaxed` 使用它只保证load和store是原子操作，其他的不做保证。

`std::memory_order_acq_rel` 相等于 release 和 acquire 的结合体。读操作遵守 acquire，写操作遵守 release。

`std::memory_order_seq_cst` 提供了顺序一致性（sequential consistency）。

> 顺序一致性保证了所有线程的执行顺序和代码中书写的保持一致。

### spin lock



## iOS中的多线程

## References

[https://stackoverflow.com/questions/46088363/why-does-stdcondition-variablewait-need-mutex](https://stackoverflow.com/questions/46088363/why-does-stdcondition-variablewait-need-mutex)

[http://blog.vladimirprus.com/2005/07/spurious-wakeups.html](http://blog.vladimirprus.com/2005/07/spurious-wakeups.html)

[https://www.codeproject.com/Articles/808305/Understand-std-atomic-](https://www.codeproject.com/Articles/808305/Understand-std-atomic-)

[https://bartoszmilewski.com/2008/12/01/c-atomics-and-memory-ordering/](https://bartoszmilewski.com/2008/12/01/c-atomics-and-memory-ordering/)

[http://www.cplusplus.com/reference/atomic/memory_order/](http://www.cplusplus.com/reference/atomic/memory_order/)