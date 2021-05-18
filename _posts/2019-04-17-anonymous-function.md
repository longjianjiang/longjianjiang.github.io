---
layout: post
title:  "匿名函数笔记"
date:   2019-04-17
excerpt:  "本文是笔者学习匿名函数的笔记"
tag:
- iOS
comments: true
---

Block是C的一个扩展，所以在C++中同样可以使用，同时C++11新增了lambda表达式，这两者其实都是匿名函数的实现。下面首先来看block。

# block

## 类型

> block其实就是一个对象，因为它有`isa`。

{% highlight cpp %}
int main() {
    auto block = ^int(int a, int b) {
        return a + b;
    };
    cout << block(5, 7) << endl;
    return 0;
}
{% endhighlight %}

上述就是一个简单的block，可以看到使用起来和函数很像，同时我们都知道block也可以作为函数参数，函数返回值。所以block是怎样一个类型呢？

block有三种类型，对应着block对象存储的不同内存位置。

```
_NSConcreteStackBlock   
_NSConcreteMallocBlock	
_NSConcreteGloalBlock
```

那么具体block里面有那些东西呢，他是怎么执行创建时候写的代码块呢？

{% highlight cpp %}
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static int __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {
    return a + b;
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main() {
    auto block = ((int (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    ((int (*)(__block_impl *, int, int))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 5, 7);
    return 0;
}
{% endhighlight %}

上面代码经过`clang -rewrite-objc xxx.m`变成了上述一大段代码。

可以看到block对象是一个结构体，结构体有:

1.有isa指针指向上述所说的三种类；

2.有一个FuncPtr指针指向了函数，这个函数就是block代码块生成的一个函数；

3.有一个desc结构体来描述block的信息;

目前基本对block有了一个新的认识，继续往下看。

## 捕获变量

所谓捕获，就是匿名函数中使用了外部定义的变量。所以根据变量的定义位置以及类型有多种可能，下面一一介绍：

### 静态区变量

静态区变量包括静态局部变量，静态变量，全局变量。

{% highlight cpp %}
int num;
void main() {
    ^{ num= 7; };
}
{% endhighlight %}

block中使用静态区变量就很简单，不会往block结构体中增加成员，修改当然也没有问题。

使用静态局部变量的时候，blcok结构体中会添加一个捕获变量类型的指针，通过指针同样可以修改静态局部变量的值。

### 局部变量

- 非对象类型

{% highlight cpp %}
struct __main_block_impl_0 {
  ...
  int num;
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int num = __cself->num; // bound by copy
  num; 
}

void main() {
    int num;
    ^{ num; };
}
{% endhighlight %}

当block中使用了一个局部变量时，此时生成的block结构体中就会加入这个局部变量作为一个新的成员，并且在block初始化的时候将外部的局部变量作为参数传递。

此时如果在block内部对局部变量进行修改是不允许的，其实道理很简单，block结构体里仅仅存了一个整形的数据，就算内部改了并不会更新外部，所以如果进行修改则直接报错。

不知道大家之前有没有注意到block根据代码块生成的一个函数，默认有一个参数`__cself`，一个指向block结构体的指针。它的作用就是为了在生成的函数中根据它来从block结构体中取到捕获的对象，如`__main_block_func_0`所示。

- 对象

{% highlight cpp %}
struct __main_block_impl_0 {
  ...
  __strong id obj;
  }
};

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->obj, (void*)src->obj, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->obj, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __main_block_desc_0 {
  ...
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
};

void main() {
  id obj = [NSMutableArray new];
	^{ [obj addObject:@1]; };
}
{% endhighlight %}

当block中使用一个对象时，block结构体同样添加了该对象，同时附上了该对象的内存修饰符`__strong`。同时描述blcok结构体的desc结构体中拥有增加了两个成员，两个函数指针，现在暂且记下，稍后笔者会解释。

跟之前的整型一样，blcok中不能修改对象，但是可以使用对象的方法，比如往数组中添加一个元素。

-  引用

之前不论是基本数据类型还是对象类型，block中捕获的这些变量都不能在内部进行修改，也就是对其赋值，如果确实需要这么做怎么办呢？

这个时候需要给外部的变量添加一个`__block` 修饰符，这个修饰符改变了捕获的策略，使用了引用，从而实现可以修改。

{% highlight cpp %}
struct __Block_byref_num_0 {
  void *__isa;
__Block_byref_num_0 *__forwarding;
 int __flags;
 int __size;
 int num;
};

struct __main_block_impl_0 {
  ...
  __Block_byref_num_0 *num; // by ref
};

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->obj, (void*)src->obj, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->obj, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  ...
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
}

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_num_0 *num = __cself->num; // bound by ref
 (num->__forwarding->num) = 7; 
}

int main() {
 __attribute__((__blocks__(byref))) __Block_byref_num_0 num = {(void*)0,(__Block_byref_num_0 *)&num, 0, sizeof(__Block_byref_num_0), 5};
 ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_num_0 *)&num, 570425344));
 printf("num = %d\n", (num.__forwarding->num));
 return 0;
}

void main() {
  __block int num = 5;
  ^{ num = 7; };
}
{% endhighlight %}

和之前捕获对象一样，描述blcok结构体的desc结构体中拥有增加了两个成员，两个函数指针。

可以看到使用`__block`后，原先的int数据变成了一个结构体 `__Block_byref_num_0`，这个结构体里包含了一个整形的成员`num`。main函数中进行初始化结构体num时，将成员`num`赋值了5。

block结构体添加了该结构体的指针，这样通过指针就可以修改结构体中的`num`的值了。`__main_block_func_0`中首先取到结构体指针，`(num->__forwarding->num) = 7; `进行修改，不过这里通过`__forwarding`这个指向自身的指针进行绕了一次有点没必要，笔者会在下面说明为什么要这么做。

可以看到使用`__block`虽然实现了可以修改，但是通过源代码可以看到是有代价的。

下面来看使用`__block`修饰对象有什么不同，代码如下:

{% highlight cpp %}
struct __Block_byref_obj_0 {
  void *__isa;
__Block_byref_obj_0 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);
 void (*__Block_byref_id_object_dispose)(void*);
 id obj;
};
{% endhighlight %}

和捕获对象一样，`__block`修饰的对象的结构体中，增加了两个成员，两个函数指针。

### 小结

block中捕获变量默认是拷贝外部变量的值，也就是说如果后面该变量修改了，block内部使用的依然是之前拷贝的那份。

所以如果需要修改外部变量，则需要使用`__block`，这样其实block内部就捕获到了外部变量的引用，也就可以实现修改了。

## 内存管理

之前已经说过block就是一个对象，既然是对象，那么就需要内存管理。之前所有的例子中的block结构体都是生成在栈上，如果将一个block表达式赋值给一个blcok类型的变量，此时这个变量指向的是栈上block结构体的地址。

我们知道block是一个匿名函数，通常将一段要执行的代码放在这个匿名函数中，可能未来某个时刻需要执行这个匿名函数，如果在执行的时候创建block的作用域已经不存在，那么栈上的所有数据都会被清空，那么接下来执行肯定就出错了。来看一个例子：

{% highlight objc %}
__unsafe_unretained blk_t block;

- (void)viewDidLoad {
    [super viewDidLoad];

    block = ^ {
        NSLog(@"name is %@", str);
    };
}

- (void)viewWillAppear:(BOOL)animated {
    [super viewWillAppear:animated];

    block(); 
}
{% endhighlight %}

这个时候viewWillAppear中执行block就会出错。那么如果需要完成这个需求怎么办呢？这个时候就需要将这个block移动到堆上，这样viewDidLoad函数返回后也不会影响viewWillAppear中执行这个block。

{% highlight cpp %}
id objc_retainBlock(id x) {
    return (id)_Block_copy(x);
}
{% endhighlight %}

在这个例子中，其实只需要将`__unsafe_unretained`去掉即可，ARC下在进行赋值之前，会调用`objc_retainBlock`将栈上的block移动到堆上，所以返回的block其实已经是`_NSConcreteMallocBlock`类型。

### _Block_copy

既然`_Block_copy`可以将栈上的block移动到堆上，下面来看一下该方法的实现。

{% highlight cpp %}
struct Block_layout {
    void *isa;
    volatile int32_t flags; // contains ref count
    int32_t reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 *descriptor;
    // imported variables
};

static int32_t latching_incr_int(volatile int32_t *where) {
    while (1) {
        int32_t old_value = *where;
        if ((old_value & BLOCK_REFCOUNT_MASK) == BLOCK_REFCOUNT_MASK) {
            return BLOCK_REFCOUNT_MASK;
        }
        if (OSAtomicCompareAndSwapInt(old_value, old_value+2, where)) {
            return old_value+2;
        }
    }
}

// 删去了关于垃圾回收的代码。
static void *_Block_copy_internal(const void *arg, const bool wantsOne) {
    struct Block_layout *aBlock;

    if (!arg) return NULL; 
    
    aBlock = (struct Block_layout *)arg;
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        return aBlock;
    }

    if (!isGC) {
        struct Block_layout *result = malloc(aBlock->descriptor->size);
        if (!result) return NULL;
        memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first
        // reset refcount
        result->flags &= ~(BLOCK_REFCOUNT_MASK|BLOCK_DEALLOCATING);    // XXX not needed
        result->flags |= BLOCK_NEEDS_FREE | 2;  // logical refcount 1
        result->isa = _NSConcreteMallocBlock;
        _Block_call_copy_helper(result, aBlock);
        return result;
    }
}

void *_Block_copy(const void *arg) {
    return _Block_copy_internal(arg, true);
}
{% endhighlight %}

实际实现在方法`_Block_copy_internal`中，实现分为以下几步:

1.首先判断block是否为空；

2.通过block结构体的flags中`BLOCK_NEEDS_FREE`位来判断该block是否已经在堆上，进行增加其引用计数；

3.如果该block是`_NSConcreteGloalBlock`类型，则直接返回；

4.此时首先根据当前block结构体的size分配一块内存；

4.1将栈上block的每一位复制到堆上内存；

4.2将block结构体中flags成员的`BLOCK_REFCOUNT_MASK`和`BLOCK_DEALLOCATING`两位设置为0；

4.3将block结构体中flags成员`BLOCK_NEEDS_FREE`位设置为1，标记是堆上block，需要释放内存；同时将`BLOCK_REFCOUNT_MASK`中最低1位设置为1;

4.4将block类型改成`_NSConcreteMallocBlock`;

5._Block_call_copy_helper方法笔者在后面进行说明；

6.最后返回堆上的block；

可以看到还是比较符合直觉的，既然有从堆上申请了内存，那么一定会有释放，对应的方法就是`_Block_release`。

### _Block_release

{% highlight cpp %}
static bool latching_decr_int_should_deallocate(volatile int32_t *where) {
    while (1) {
        int32_t old_value = *where;
        if ((old_value & BLOCK_REFCOUNT_MASK) == BLOCK_REFCOUNT_MASK) {
            return false; // latched high
        }
        if ((old_value & BLOCK_REFCOUNT_MASK) == 0) {
            return false;   // underflow, latch low
        }
        int32_t new_value = old_value - 2;
        bool result = false;
        if ((old_value & (BLOCK_REFCOUNT_MASK|BLOCK_DEALLOCATING)) == 2) {
            new_value = old_value - 1;
            result = true;
        }
        if (OSAtomicCompareAndSwapInt(old_value, new_value, where)) {
            return result;
        }
    }
}

static void (*_Block_deallocator)(const void *) = (void (*)(const void *))free;
static void _Block_destructInstance_default(const void *aBlock) {}

void _Block_release(const void *arg) {
    struct Block_layout *aBlock = (struct Block_layout *)arg;
    if (!aBlock 
        || (aBlock->flags & BLOCK_IS_GLOBAL)
        || ((aBlock->flags & (BLOCK_IS_GC|BLOCK_NEEDS_FREE)) == 0)
        ) return;
    else if (aBlock->flags & BLOCK_NEEDS_FREE) {
        if (latching_decr_int_should_deallocate(&aBlock->flags)) {
            _Block_call_dispose_helper(aBlock);
            _Block_destructInstance(aBlock);
            _Block_deallocator(aBlock);
        }
    }
}
{% endhighlight %}

`_Block_release`实现更加简单，进行类型转换后，如果结构体中flags成员`BLOCK_NEEDS_FREE`位设置为1，那么则进行一次引用计数的减少，如果返回需要释放，则调用方法进行释放。

`_Block_destructInstance`调用的其实是`_Block_destructInstance_default`，该方法啥也没干，而`_Block_deallocator`则就是`free`方法，也就是释放内存。

`_Block_call_dispose_helper` 下面进行说明。

### _Block_call_copy_helper & _Block_call_dispose_helper

首先给出block的结构图:

![anonymous_function_1]({{site.url}}/assets/images/blog/anonymous_function_1.png)

可以看到描述block的结构体desc中有之前提到过的两个函数指针。而且在block中捕获基本数据类型的时候是没有的，当捕获的是对象类型或者是`__block`修饰的类型时就有了，所以很容易猜到这两个方法是用来管理block中捕获的对象的内存的。

这也就解释了为什么说block中持有捕获的对象，会延长该对象的生命周期，正因为是这样所以才导致了block使用不当会造成循环引用导致内存泄露。

先给出`_Block_call_copy_helper`的实现:

{% highlight cpp %}
struct Block_descriptor_2 {
    // requires BLOCK_HAS_COPY_DISPOSE
    void (*copy)(void *dst, const void *src);
    void (*dispose)(const void *);
};
static void _Block_call_copy_helper(void *result, struct Block_layout *aBlock) {
    struct Block_descriptor_2 *desc = _Block_descriptor_2(aBlock);
    if (!desc) return;

    (*desc->copy)(result, aBlock); // do fixup
}
{% endhighlight %}

将block结构体尝试移动改变为`Block_descriptor_2`类型，调用其中的copy函数指针，也就是类似如下的一个方法:

{% highlight cpp %}
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
  _Block_object_assign((void*)&dst->obj, (void*)src->obj, 3/*BLOCK_FIELD_IS_OBJECT*/);
}
{% endhighlight %}

`__main_block_copy_0`中会对block中捕获的所有对象进行调用`_Block_object_assign`方法。下面继续看`_Block_object_assign`的实现：

{% highlight cpp %}
enum {
    BLOCK_FIELD_IS_OBJECT   =  3,  // id, NSObject, __attribute__((NSObject)), block, ...
    BLOCK_FIELD_IS_BLOCK    =  7,  // a block variable
    BLOCK_FIELD_IS_BYREF    =  8,  // the on stack structure holding the __block variable
    BLOCK_FIELD_IS_WEAK     = 16,  // declared __weak, only used in byref copy helpers
    BLOCK_BYREF_CALLER      = 128, // called from __block (byref) copy/dispose support routines.
};

static void _Block_assign_default(void *value, void **destptr) {
    *destptr = value;
}
static void (*_Block_assign)(void *value, void **destptr) = _Block_assign_default;

void _Block_object_assign(void *destAddr, const void *object, const int flags) {
    switch (os_assumes(flags & BLOCK_ALL_COPY_DISPOSE_FLAGS)) {
      case BLOCK_FIELD_IS_OBJECT:
        /*******
        id object = ...;
        [^{ object; } copy];
        ********/
            
        _Block_retain_object(object);
        _Block_assign((void *)object, destAddr);
        break;

      case BLOCK_FIELD_IS_BLOCK:
        /*******
        void (^object)(void) = ...;
        [^{ object; } copy];
        ********/

        _Block_assign(_Block_copy_internal(object, false), destAddr);
        break;
    
      case BLOCK_FIELD_IS_BYREF | BLOCK_FIELD_IS_WEAK:
      case BLOCK_FIELD_IS_BYREF:
        /*******
         // copy the onstack __block container to the heap
         __block ... x;
         __weak __block ... x;
         [^{ x; } copy];
         ********/
        
        _Block_byref_assign_copy(destAddr, object, flags);
        break;
        
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK:
        /*******
         // copy the actual field held in the __block container
         __block id object;
         __block void (^object)(void);
         [^{ object; } copy];
         ********/

        // under manual retain release __block object/block variables are dangling
        _Block_assign((void *)object, destAddr);
        break;

      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT | BLOCK_FIELD_IS_WEAK:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK  | BLOCK_FIELD_IS_WEAK:
        /*******
         // copy the actual field held in the __block container
         __weak __block id object;
         __weak __block void (^object)(void);
         [^{ object; } copy];
         ********/

        _Block_assign_weak(object, destAddr);
        break;

      default:
        break;
    }
}
{% endhighlight %}

根据flags枚举的不同取值组合，分为如下几类：

- 对象类型

对对象调用了`_Block_retain_object`，其实就是`retain`方法，也就是说这里就是完成了所谓的block持有外部变量。

_Block_assign简单的将栈上block捕获的对象，通过指针赋值到堆上新生成block中的捕获对象。

- block类型

首先将block中捕获的栈上的block调用`_Block_copy_internal`移动到堆上，依然使用_Block_assign简单的将栈上block捕获的对象，通过指针赋值到堆上新生成block中的捕获对象。

也就是所有的block都会被移动到堆上。

- __block类型

__block类型处理起来多了一些步骤，还记得之前我们看到__block修饰对象的时候，生成的byref结构体中依然有copy&dispose函数指针。

因为byref结构体默认还是在栈上，所以当block移动到堆上时还需要将byref移动过到堆上，操作类似block的copy，同样的也需要对byref结构体中的对象进行内存管理，使用的就是byref结构体中copy&dispose函数指针。

下面来看`_Block_byref_assign_copy`的实现:

{% highlight cpp %}
struct Block_byref {
    void *isa;
    struct Block_byref *forwarding;
    volatile int32_t flags; // contains ref count
    uint32_t size;
};

struct Block_byref_2 {
    // requires BLOCK_BYREF_HAS_COPY_DISPOSE
    void (*byref_keep)(struct Block_byref *dst, struct Block_byref *src);
    void (*byref_destroy)(struct Block_byref *);
};

static void _Block_byref_assign_copy(void *dest, const void *arg, const int flags) {
    struct Block_byref **destp = (struct Block_byref **)dest;
    struct Block_byref *src = (struct Block_byref *)arg;
        
    if (src->forwarding->flags & BLOCK_BYREF_IS_GC) {
        ;   // don't need to do any more work
    }
    else if ((src->forwarding->flags & BLOCK_REFCOUNT_MASK) == 0) {
        // src points to stack
        bool isWeak = ((flags & (BLOCK_FIELD_IS_BYREF|BLOCK_FIELD_IS_WEAK)) == (BLOCK_FIELD_IS_BYREF|BLOCK_FIELD_IS_WEAK));
        // if its weak ask for an object (only matters under GC)
        struct Block_byref *copy = (struct Block_byref *)_Block_allocator(src->size, false, isWeak);
        copy->flags = src->flags | _Byref_flag_initial_value; // non-GC one for caller, one for stack
        copy->forwarding = copy; // patch heap copy to point to itself (skip write-barrier)
        src->forwarding = copy;  // patch stack to point to heap copy
        copy->size = src->size;
        if (isWeak) {
            copy->isa = &_NSConcreteWeakBlockVariable;  // mark isa field so it gets weak scanning
        }
        if (src->flags & BLOCK_BYREF_HAS_COPY_DISPOSE) {
            // Trust copy helper to copy everything of interest
            // If more than one field shows up in a byref block this is wrong XXX
            struct Block_byref_2 *src2 = (struct Block_byref_2 *)(src+1);
            struct Block_byref_2 *copy2 = (struct Block_byref_2 *)(copy+1);
            copy2->byref_keep = src2->byref_keep;
            copy2->byref_destroy = src2->byref_destroy;

            if (src->flags & BLOCK_BYREF_LAYOUT_EXTENDED) {
                struct Block_byref_3 *src3 = (struct Block_byref_3 *)(src2+1);
                struct Block_byref_3 *copy3 = (struct Block_byref_3*)(copy2+1);
                copy3->layout = src3->layout;
            }

            (*src2->byref_keep)(copy, src);
        }
        else {
            // just bits.  Blast 'em using _Block_memmove in case they're __strong
            // This copy includes Block_byref_3, if any.
            _Block_memmove(copy+1, src+1,
                           src->size - sizeof(struct Block_byref));
        }
    }
    // already copied to heap
    else if ((src->forwarding->flags & BLOCK_BYREF_NEEDS_FREE) == BLOCK_BYREF_NEEDS_FREE) {
        latching_incr_int(&src->forwarding->flags);
    }
    // assign byref data block pointer into new Block
    _Block_assign(src->forwarding, (void **)destp);
}
{% endhighlight %}

这里就可以解释为什么byref结构体中有一个`forwarding`指针指向自身了，因为当byref移动到堆时，原来栈上byref结构体的`forwarding`指向了堆上byref结构体。这样不论byref在栈上还是在堆上都可以通过`obj->__forwarding->obj`在block中访问到byref结构体变量。

下面来看这个方法的工作流程:

1.参数类型转换，转成了`Block_byref`结构体类型；

2.byref结构体flags中存储捕获对象引用计数的位为0，此时byref结构体在栈上，进行移动到堆上；

2.1堆上分配内存创建一个byref结构体，将栈上byref结构体中的信息拷贝到堆上生成的byref结构体；

2.2将堆上byref结构体flags`BLOCK_BYREF_NEEDS_FREE`设置为1，`BLOCK_REFCOUNT_MASK`设置为2；

2.3将栈上byref结构体的`forwarding`指向了堆上byref结构体；

2.4如果byref中结构体中捕获成员是对象类型，则将copy&dispose指针信息进行复制；

2.5执行函数指针指向的函数，其实还是调用了`_Block_object_assign`方法，只是参数和flags不同了，也就是类似如下的一个方法:   

> 可以看到就是移动指针到byref结构体中捕获成员的位置，同时flags中按位于了`BLOCK_BYREF_CALLER`。

{% highlight cpp %}
static void __Block_byref_id_object_copy_131(void *dst, void *src) {
 _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}
{% endhighlight %}

2.6如果byref中结构体中捕获成员不是对象类型，则简单的使用`memmove`将byref结构体中其他信息按位复制到堆上的byref结构体；

3.byref结构体已经在堆上，增加引用计数；

4._Block_assign将堆上新生成的byref结构体通过指针赋值到堆上新生成block中的捕获的byref结构体。

过程如下图所示：

![anonymous_function_2]({{site.url}}/assets/images/blog/anonymous_function_2.png)

- __block 中 对象类型/__block 中 __weak 对象类型

将obj通过指针赋值到desc指向的对象。

---

至此`_Block_object_assign`结束。下面继续看`_Block_object_dispose`，当block对象销毁时，会调用`Block_descriptor_2`中的dispose函数指针，也就是类似如下的方法:

{% highlight cpp %}
static void __main_block_dispose_0(struct __main_block_impl_0*src) {
  _Block_object_dispose((void*)src->obj, 8/*BLOCK_FIELD_IS_BYREF*/);
}
{% endhighlight %}

下面给出`_Block_object_dispose`的实现：

{% highlight cpp %}
static void _Block_byref_release(const void *arg) {
    struct Block_byref *byref = (struct Block_byref *)arg;
    int32_t refcount;

    byref = byref->forwarding;
    
    if ((byref->flags & BLOCK_BYREF_NEEDS_FREE) == 0) {
        return; // stack or GC or global
    }
    refcount = byref->flags & BLOCK_REFCOUNT_MASK;
  os_assert(refcount);
    if (latching_decr_int_should_deallocate(&byref->flags)) {
        if (byref->flags & BLOCK_BYREF_HAS_COPY_DISPOSE) {
            struct Block_byref_2 *byref2 = (struct Block_byref_2 *)(byref+1);
            (*byref2->byref_destroy)(byref);
        }
        _Block_deallocator((struct Block_layout *)byref);
    }
}

void _Block_object_dispose(const void *object, const int flags) {
    switch (os_assumes(flags & BLOCK_ALL_COPY_DISPOSE_FLAGS)) {
      case BLOCK_FIELD_IS_BYREF | BLOCK_FIELD_IS_WEAK:
      case BLOCK_FIELD_IS_BYREF:
        // get rid of the __block data structure held in a Block
        _Block_byref_release(object);
        break;
      case BLOCK_FIELD_IS_BLOCK:
        _Block_destroy(object);
        break;
      case BLOCK_FIELD_IS_OBJECT:
        _Block_release_object(object);
        break;
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_OBJECT | BLOCK_FIELD_IS_WEAK:
      case BLOCK_BYREF_CALLER | BLOCK_FIELD_IS_BLOCK  | BLOCK_FIELD_IS_WEAK:
        break;
      default:
        break;
    }
}
{% endhighlight %}

可以看到相比assign，dispose简单不少。依然分为以下几种情况：

1.__block类型

调用了`_Block_byref_release`方法，该方法和block的dispose实现类似，同时会尝试调用byref结构体中的dispose函数指针，类似下面一个方法:

{% highlight cpp %}
static void __Block_byref_id_object_dispose_131(void *src) {
 _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}
{% endhighlight %}

此时就到了第4种类型；

2.block类型

销毁block对象，`_Block_destroy` 方法内部会调用`_Block_release`，也就是要重新走一遍dispose的流程；

3.对象类型

将对象进行一次release；

4.__block 中 对象类型/__block 中 __weak 对象类型

什么也不干，因为assign的时候，只是简单的赋值。

### 循环引用

这里来说下block和循环引用，看一个例子:

{% highlight objc %}
- (void)foo {
    __weak typeof(self) weakSelf = self;
    self.block = ^{
        // 注释1
        __strong typeof(weakSelf) strongSelf = weakSelf;

        /* 注释2
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            NSLog(@"%@",strongSelf);
        });
        */
    };
    self.block();
    [self.navigationController popViewControllerAnimated:YES];
}
{% endhighlight %}

笔者先提几个问题：

1.block中使用弱引用的self，为什么不会导致循环引用？

`_Block_object_assign`中当捕获的对象是对象类型，那么会进行一次retain操作。如果在一个持有block中使用了对象本身，那么便会形成循环引用。

解决办法其实也很简单将对象的弱引用放到block中使用。所以可以肯定的是当是弱引用的时候，`_Block_retain_object(object);`并没有增加弱引用指向对象的引用计数，否则依然还会出现循环引用。

不过笔者做过实验，对一个弱引用进行retain操作，会调用到`objc_loadWeakRetained`，会增加弱引用指向对象的引用计数。

所以笔者猜测`_Block_retain_object`内部判断捕获对象如果是`__strong`则增加引用计数，如果是`__weak`则不增加引用计数。

2.为什么有时候需要加注释1这段代码？

因为weakSelf指向的对象可能会在执行block时候已经销毁了。

注释1这段代码会调用`objc_loadWeakRetained`，会将弱引用指向的对象的引用计数加1，也就是为了防止弱引用指向的对象释放。

3.注释1这段代码中strongSelf为什么明确的使用`__strong`，不是默认使用的就是`__strong`吗？

因为`typeof(weakSelf)`的修饰符不是`__strong`，所以需要明确指定。

下面笔者来说下这个例子:

上面很重要的一点是strongSelf是一个局部变量，也就是说strongSelf虽然让控制器引用计数加1，但是当strongSelf过了block的作用域，就会调用`objc_storeStrong`对控制器引用计数减1。

但是dispatch_after的参数又是一个block，block中使用到了strongSelf，所以会对strongSelf进行一次retain，等于对控制器引用计数又加了1。当strongSelf超过block作用域会进行一次release，此时控制器引用计数减1不为0，所以还没有释放，等3s后dispatch_afterblock执行完，也就销毁了该block，此时会strongSelf进行一次release，此时控制器减1为0，完成释放。

每次都写这两段重复代码，而且block中使用了strongSelf，所以RAC中提供了宏来帮我们自动生成，具体查看[这里](https://ziecho.com/post/ios/2016-08-04)

### 引用计数

最后笔者想说下block变量的引用计数，根据之前的分析，我们知道block的引用计数存储在了结构体中flags中。所以通过`_objc_rootRetainCount`来查看block变量的引用计数返回的都是1。下面看一个例子:

{% highlight cpp %}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        id sark = [Sark new];
        blk_t blk = ^{ sark; };
        blk_t ref = blk;

        [blk copy];

        struct Block_layout *aBlock;
        aBlock = (__bridge struct Block_layout *)blk;
        int refCount = aBlock->flags & 0xfffe;
        int count = _objc_rootRetainCount(blk);

        NSLog(@"count = %d, refCount = %d", count, refCount);
    }
    return 0;
}

// count = 1, refCount = 4
// sark dealloc
{% endhighlight %}

可以看到refCount是4，因为每次`_Block_copy_internal`内部增加引用计数每次增加2，所以实际引用计数为2，因为blk和ref都指向block结构体，所以是没问题的。

不过我们发现调用`[blk copy];`并没有效果，并没有增加block的引用计数。

当作用域结束，blk和ref销毁，指向的block结构体也被销毁，所以block内部捕获的sark也释放，从而调用了dealloc方法。

继续看另一种情况：

{% highlight cpp %}
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        id sark = [Sark new];
        blk_t blk = ^{ sark; };
        blk_t ref = blk;

        _Block_copy((__bridge const void *)(blk));
        _Block_copy((__bridge const void *)(blk));

        struct Block_layout *aBlock;
        aBlock = (__bridge struct Block_layout *)blk;
        int refCount = aBlock->flags & 0xfffe;
        int count = _objc_rootRetainCount(blk);

        NSLog(@"count = %d, refCount = %d", count, refCount);

        _Block_release((__bridge const void *)(blk));
        refCount = aBlock->flags & 0xfffe;
        NSLog(@"count = %d, refCount = %d", count, refCount);
    }
    return 0;
}

// count = 1, refCount = 8
// count = 1, refCount = 6
{% endhighlight %}

这次我们调用了两次`_Block_copy`方法，可以修改block的引用计数，之后调用了一次`_Block_release`，block的引用计数确实减1了。

当作用域结束，block引用计数不为0，所以block没有释放，导致了捕获的sark也没有释放。

通过这两个例子可以发现，当我们对一个block对象进行copy方法时，如果此时block已经在堆上，那么此时啥也没干，除非直接调用`_Block_copy`才会增加引用计数。猜测这样做应该是为了防止意外copy导致内存泄露。

### 对比函数指针

一般block作为回调使用，函数指针也可以实现同样的功能。所以他们的区别有哪些呢？

笔者认为一个最大的区别在于，block可以捕获上下文的相关变量，动态构造出一个结构体，如果想要改变变量，声明为`__block`即可。而函数指针则不具备这样的能力。
# lambda

上面花了很长的篇幅来介绍block的实现，现在让我们简单看下C++11中的lambda表达式。

{% highlight cpp %}
int x = 5, y = 7;

auto lamdba = [](int x, int y) -> int {
    return x + y;
};

cout << "sum = " << lamdba(x, y) << endl;
{% endhighlight %}

上面就是一个lambda表达式，大致结构和block类似，最前面的`[]`是用来指定捕获变量及捕获方式，具体规则见下表:

```
[]：默认不捕获任何变量；
[=]：默认以值捕获所有变量；
[&]：默认以引用捕获所有变量；
[x]：仅以值捕获x，其它变量不捕获；
[&x]：仅以引用捕获x，其它变量不捕获；
[=, &x]：默认以值捕获所有变量，但是x是例外，通过引用捕获；
[&, x]：默认以引用捕获所有变量，但是x是例外，通过值捕获；
[this]：通过引用捕获当前对象（其实是复制指针）；
[*this]：通过传值方式捕获当前对象；
```

即使一个局部变量以值的方式进行捕获，我们想在lambda中修改该值，只需要加上`mutable`关键字即可修改，如下例子所示:

{% highlight cpp %}
int x = 5, y = 7;

auto lamdba = [](int x, int y) mutable -> int {
    x++; y++;
    return x + y;
};

cout << "x = " << x << ", y = " << y << endl; // x = 5, y = 7
cout << "sum = " << lamdba(x, y) << endl; // sum = 14
{% endhighlight %}

其实lambda的实现是一个匿名类，重载了`()`运算符，如果捕获了对象，那么这些对象会作为这个匿名类的成员。

lambda同样会在适当的时候进行移动到堆上，因为如果不移动过了作用域，接下来执行的时候肯定就出错了，下面看一个例子:

{% highlight cpp %}
struct Inner {
    int v;
} in{100};

struct Outer {
    int v;
    Inner *t;
};

function<void()> getLambda() {
    Outer out{0, &in};
    return [out]() mutable {
        cout << out.v++ << "\t" << out.t->v++ << endl;
    };
}

int main() {
    auto lambda = getLambda();
    auto copy_lambda = lambda;
    lambda();
    lambda();
    copy_lambda();
    copy_lambda();

    return 0;
}

// output
0 100
1 101
0 102
1 103
{% endhighlight %}

可以发现当函数返回的时候，lambda被移动到了堆上，同时所捕获的对象也跟着移动到了堆。

当lamdba进行复制的时候，会拷贝匿名类中的成员，当成员是指针类型，进行的是浅拷贝，并没有重复分配内存空间，所以打印结构体`in`的值时依然是叠加的。

lambda的更多内容可以参考[这里](https://www.jianshu.com/p/d686ad9de817)

# 总结

本文主要分析了block的实现结构，block变量以及其捕获局部变量内存管理的实现，最后简单介绍了C++11中的lambda。

# References

[iOS与OS X多线程和内存管理](https://book.douban.com/subject/24720270/)

[https://www.mikeash.com/pyblog/friday-qa-2011-06-03-objective-c-blocks-vs-c0x-lambdas-fight.html](https://www.mikeash.com/pyblog/friday-qa-2011-06-03-objective-c-blocks-vs-c0x-lambdas-fight.html)

[http://www.galloway.me.uk/2013/05/a-look-inside-blocks-episode-3-block-copy/](http://www.galloway.me.uk/2013/05/a-look-inside-blocks-episode-3-block-copy/)

[https://github.com/apple/swift-corelibs-foundation](https://github.com/apple/swift-corelibs-foundation)

[https://ziecho.com/post/ios/2015-09-02](https://ziecho.com/post/ios/2015-09-02)

[https://blog.yiz96.com/cpp-closure/](https://blog.yiz96.com/cpp-closure/)
