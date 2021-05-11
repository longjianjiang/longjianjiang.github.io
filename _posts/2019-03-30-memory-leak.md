
# 内存泄漏

所谓内存泄露是指程序运行中，使用malloc/calloc之类的函数申请的内存，使用完后没有调用free进行释放，导致这块内存不能被重复利用。

在iOS开发中，内存泄露一般是以下的几种情况：

1> 循环引用；

2> 对象被全局对象持有或间接持有，导致不能释放；

3> core fundation 或者 非OC自动管理的对象，未调用释放方法；

# Instruments 检测

用Instruments的 `leaks` , `allocation`，可以很精确的查看内存问题。

## leaks

leaks可以用来检测堆上内存是否泄露，检测的原理是判断堆上的某个内存是否存在指针指向，如果没有指针指向某块内存，此时会判定为泄露。

不过当循环引用的对象，被全局指针，比如static的void *直接/间接指向，虽然不会改变引用计数，但是此时leaks是检测不到的，因为此时堆上内存是有指针指向的，是可达到的。

## Allocations

Allocations 主要统计的是All Heap & Anonymous VM的内存使用量。

[Debug Navigator](https://www.jianshu.com/p/827996b7aed0)其实就是统计了当前进程的所有虚拟内存的Dirty Size + Swapped Size，包含了_DATA数据段，Stack（函数栈）等等Allocations没有统计的内存区域。

All Heap就是App运行过程中在堆上分配的内存。

为了更好的管理内存页，系统将一组连续的内存页关联到一个VMObject上，也就是VM Region。我们在Instruments的Anonymous VM里看到的每条记录都是一个VMObject。

堆区会被划分成很多不同的VM Region，不同类型的内存分配根据需求进入不同的VM Region。有MALLOC_LARGE，MALLOC_SMALL，还有MALLOC_TINY， MALLOC metadata等等。划分出这些不同的大小目的是减少内存碎片的产生。

### 内存类型

Dirty Memory: 可写的内存，内存紧张时可以被回收。

Swapped Memory: 被置换到磁盘的内存。

Compressed Memory: 由于iOS中没有memory swap技术，无法时用硬盘资源来置换内存，因此当资源紧张时，需要新的手段来提高内存资源利用率。Compress Memory是iOS 7之后引入的内存压缩技术，它可以将没有被访问的内存对象所在的Page进行压缩，从而腾出更多的空间。如果程序在某个时刻要访问被压缩的对象，则再将该对象所在的Page进行Decompression。

Resident Memory: 进程的virtual memory中常驻在物理RAM中的部分。

Clean Memory: 只读的内存，比如被加载到内存中的应用程序的__TEXT段, 系统的各种framework代码和资源。内存紧张的时候会被回收，然后重新创建。

Private和Shared Memroy

RAM中可以被多个进程共享的部分称为Shared Memory，比如系统的framework，它只映射一份代码到内存，这部分内存会被不同的进程共用。而每个进程单独alloc的内存，则是Private Memory。

### 内存警告

当我们的app在前台运行时，不断消耗内存，导致内存不足时，系统首先会将不用的clean memroy干掉一部分，腾出空间来继续创建drity memory，当dirty memory越来越多，又导致内存不足时，系统会将运行在后台app的dirty memory干掉，然后将之前干掉的clean memory重新load回来。

### nszone malloc_zone_t

所谓zone就是一组内存块，在某个Zone里分配的内存块，会随着这个Zone的销毁而销毁，所以Zone可以加速大量小内存块的集体销毁。

```
void allocCustomObjectsWithCustomMallocZone() {
    malloc_zone_t *customZone = malloc_create_zone(1024, 0);
    malloc_set_zone_name(customZone, "custom malloc zone");
    for (int i = 0; i < 1000; ++i) {
        malloc_zone_malloc(customZone, 300 * 4096);
    }
    malloc_destroy_zone(customZone)
}
```

# VM Tracker

[ref](https://www.jianshu.com/p/f82e2b378455)

# References

[ref1](https://github.com/facebook/FBRetainCycleDetector)

[ref2](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/FindingLeaks.html#//apple_ref/doc/uid/20001883-SW2)

[ref3](http://www.cocoachina.com/articles/16951)

[ref4](https://triplecc.github.io/2019/07/15/%E6%90%AD%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E6%A3%80%E6%B5%8B%E5%B7%A5%E5%85%B7/)

[leaks](https://www.jianshu.com/p/12cadd05e370)

[memory](https://squarepants1991.github.io/ios%E5%BC%80%E5%8F%91/2018/01/16/%E6%8E%A2%E7%B4%A2iOS%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D.html)
