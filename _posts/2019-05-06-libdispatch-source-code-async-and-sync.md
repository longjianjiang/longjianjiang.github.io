---
layout: post
title:  "【libdispatch 源码】async & sync"
date:   2019-05-06
excerpt:  "本文是笔者阅读libdispatch源码关于异步执行以及同步执行的笔记"
tag:
- libdispatch
comments: true
---

## async

`dispatch_async`应该是使用最频繁的API了，下面笔者将去查看其源码实现。

{% highlight cpp %}
void dispatch_async(dispatch_queue_t queue, 
	dispatch_block_t block);

void dispatch_async_f(dispatch_queue_t queue,
	void *_Nullable context, dispatch_function_t work);
{% endhighlight %}

上面就是GCD提供的两个异步的API，我们一般使用的是block的那个，下面来看下两个方法的实现：

{% highlight cpp %}
void dispatch_async(dispatch_queue_t dq, dispatch_block_t work) {
	dispatch_continuation_t dc = _dispatch_continuation_alloc();
	uintptr_t dc_flags = DC_FLAG_CONSUME;
	dispatch_qos_t qos;

	qos = _dispatch_continuation_init(dc, dq, work, 0, dc_flags);
	_dispatch_continuation_async(dq, dc, qos, dc->dc_flags);
}

void _dispatch_async_f(dispatch_queue_t dq, void *ctxt, dispatch_function_t func,
		dispatch_block_flags_t flags) {
	dispatch_continuation_t dc = _dispatch_continuation_alloc_cacheonly();
	uintptr_t dc_flags = DC_FLAG_CONSUME;
	dispatch_qos_t qos;

	qos = _dispatch_continuation_init_f(dc, dq, ctxt, func, flags, dc_flags);
	_dispatch_continuation_async(dq, dc, qos, dc->dc_flags);
}

void dispatch_async_f(dispatch_queue_t dq, void *ctxt, dispatch_function_t func) {
	_dispatch_async_f(dq, ctxt, func, 0);
}
{% endhighlight %}


## References

{% highlight cpp %}
{% endhighlight %}