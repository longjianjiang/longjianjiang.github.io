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

## 非阻塞型同步

- atomic

- spin lock

## 阻塞型同步

- mutex

- lock

- conditoin_variable

## iOS中的多线程
