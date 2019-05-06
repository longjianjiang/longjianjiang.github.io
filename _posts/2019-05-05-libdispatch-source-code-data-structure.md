---
layout: post
title:  "【libdispatch 源码】数据结构"
date:   2019-05-05
excerpt:  "本文是笔者阅读libdispatch源码关于其中数据结构的笔记"
tag:
- libdispatch
comments: true
---

> 下面所说的数据结构很多有两种，一种是`_s`结尾表示是结构体，一种是`_t`结尾表示是`_s`的指针形式。

## 简单类型

- dispatch_qos_t & dispatch_priority_t

{% highlight cpp %}
typedef uint32_t dispatch_qos_t;
typedef uint32_t dispatch_priority_t;
{% endhighlight %}

这两个就是整形的别名。

- dispatch_function_t

{% highlight cpp %}
typedef void (*dispatch_function_t)(void *_Nullable);
{% endhighlight %}

这个类型就是一个函数指针。

- dispatch_block_t

{% highlight cpp %}
typedef void (^dispatch_block_t)(void);
{% endhighlight %}

这个类型就是一个空参数列表无返回值的block类型。

## dispatch_continuation_t

{% highlight cpp %}
typedef struct dispatch_continuation_s {
	DISPATCH_CONTINUATION_HEADER(continuation);
} *dispatch_continuation_t;
{% endhighlight %}

这个结构体在GCD对外的API中没有暴露，是用来包装执行函数的，具体成员如下:

{% highlight cpp %}
#define DISPATCH_CONTINUATION_HEADER(x) \
	union { \
		const void *do_vtable; \
		uintptr_t dc_flags; \
	}; \
	union { \
		pthread_priority_t dc_priority; \
		int dc_cache_cnt; \
		uintptr_t dc_pad; \
	}; \
	struct dispatch_##x##_s *volatile do_next; \
	struct voucher_s *dc_voucher; \
	dispatch_function_t dc_func; \
	void *dc_ctxt; \
	void *dc_data; \
	void *dc_other
{% endhighlight %}

`dc_func`成员就是队列中需要执行任务的函数，而`dc_ctxt`则是作为函数的参数。

## queue

GCD实现了一个队列的类簇，如下所示：

```
 dispatch_queue_t
  +--> dispatch_workloop_t
  +--> dispatch_queue_serial_t --> dispatch_queue_main_t
  +--> dispatch_queue_concurrent_t
  '--> dispatch_queue_global_t
```

上面的类型都是指针类型，对应的具体的结构体类型，种类就更多更细了，如下所示：

```
  dispatch_queue_class_t / struct dispatch_queue_s
   +--> struct dispatch_workloop_s
   '--> dispatch_lane_class_t
         +--> struct dispatch_lane_s
         |     +--> struct dispatch_source_s
         |     '--> struct dispatch_mach_s
         +--> struct dispatch_queue_static_s
         '--> struct dispatch_queue_global_s
               +--> struct dispatch_queue_pthread_root_s
```


这个队列类型是我们所能接触到的，下面来看其组成：

{% highlight cpp %}
#define DISPATCH_ATOMIC64_ALIGN  __attribute__((aligned(8)))

typedef struct dispatch_queue_s *dispatch_queue_t;

struct dispatch_queue_s {
	DISPATCH_QUEUE_CLASS_HEADER(queue, void *__dq_opaque1);
} DISPATCH_ATOMIC64_ALIGN;
{% endhighlight %}

结构体成员依然用了一个宏来定义，看这个宏:

{% highlight cpp %}
#define DISPATCH_QUEUE_CLASS_HEADER(x, __pointer_sized_field__) \
	_DISPATCH_QUEUE_CLASS_HEADER(x, __pointer_sized_field__); \
	/* LP64 global queue cacheline boundary */ \
	unsigned long dq_serialnum; \
	const char *dq_label; \
	DISPATCH_UNION_LE(uint32_t volatile dq_atomic_flags, \
		const uint16_t dq_width, \
		const uint16_t __dq_opaque2 \
	); \
	dispatch_priority_t dq_priority; \
	union { \
		struct dispatch_queue_specific_head_s *dq_specific_head; \
		struct dispatch_source_refs_s *ds_refs; \
		struct dispatch_timer_source_refs_s *ds_timer_refs; \
		struct dispatch_mach_recv_refs_s *dm_recv_refs; \
	}; \
	int volatile dq_sref_cnt
{% endhighlight %}

这里我们看到了一些熟悉的参数，比如`dq_label`标示队列的名字，`dq_priority`标示队列的优先级。

成员还没有结束，内部还使用了其他宏，继续看:

{% highlight cpp %}
#define _DISPATCH_QUEUE_CLASS_HEADER(x, __pointer_sized_field__) \
	DISPATCH_OBJECT_HEADER(x); \
	__pointer_sized_field__; \
	DISPATCH_UNION_LE(uint64_t volatile dq_state, \
			dispatch_lock dq_state_lock, \
			uint32_t dq_state_bits \
	)
{% endhighlight %}

`DISPATCH_UNION_LE`宏使用的宏比较多，这里就不贴代码了，具体代码可以查看[这里](https://github.com/apple/swift-corelibs-libdispatch/blob/master/src/internal.h#L385-L408)，这个宏的作用就是生成一个下面的联合:

{% highlight cpp %}
union { 
	uint32_t volatile dq_atomic_flags; 
	struct { 
		const uint16_t dq_width; 
		const uint16_t __dq_opaque2; 
	}; 
};
{% endhighlight %}

还没有结束，`DISPATCH_OBJECT_HEADER`依然是一个宏，继续往下看：

{% highlight cpp %}
#define _DISPATCH_OBJECT_HEADER(x) \
	struct _os_object_s _as_os_obj[0]; \
	OS_OBJECT_STRUCT_HEADER(dispatch_##x); \
	struct dispatch_##x##_s *volatile do_next; \
	struct dispatch_queue_s *do_targetq; \
	void *do_ctxt; \
	void *do_finalizer

#define DISPATCH_OBJECT_HEADER(x) \
	struct dispatch_object_s _as_do[0]; \
	_DISPATCH_OBJECT_HEADER(x)
{% endhighlight %}

此时看到一个在`dispatch_continuation_t`中出现过的`do_next`成员。

再看最后一个`OS_OBJECT_STRUCT_HEADER`宏的实现：

{% highlight cpp %}
#define _OS_OBJECT_HEADER(isa, ref_cnt, xref_cnt) \
        isa; /* must be pointer-sized */ \
        int volatile ref_cnt; \
        int volatile xref_cnt

#define OS_OBJECT_STRUCT_HEADER(x) \
	_OS_OBJECT_HEADER(\
	const struct x##_vtable_s *do_vtable, \
	do_ref_cnt, \
	do_xref_cnt)
{% endhighlight %}

`_OS_OBJECT_HEADER`定一个了一个对象的结构体，因为看到了熟悉的isa，以及引用计数。

至此队列结构体完毕，可以看到还是十分复杂的一个结构。

## References

{% highlight cpp %}
{% endhighlight %}

