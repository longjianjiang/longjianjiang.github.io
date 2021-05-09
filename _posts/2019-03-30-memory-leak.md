
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
# 其他工具

# References

[ref1](https://github.com/facebook/FBRetainCycleDetector)

[ref2](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/FindingLeaks.html#//apple_ref/doc/uid/20001883-SW2)

[ref3](http://www.cocoachina.com/articles/16951)

[ref4](https://triplecc.github.io/2019/07/15/%E6%90%AD%E5%BB%BA%E8%87%AA%E5%B7%B1%E7%9A%84%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E6%A3%80%E6%B5%8B%E5%B7%A5%E5%85%B7/)

[leaks](https://www.jianshu.com/p/12cadd05e370)
