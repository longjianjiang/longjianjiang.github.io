
app 启动优化

# 启动过程

大致过程如下：

				T1                        T2
点击app图标 -----------> 执行main函数 -----------> didFinishLaunchingWithOptions

T1指的是main函数执行之前，这个时候操作系统将可执行文件加载进内存，执行一些加载&链接工作，最后开始执行main函数。
T2指的是main函数执行之后，到didFinishLaunchingWithOptions执行完毕这一段时间。
T3指的是didFinishLaunchingWithOptions返回后，此时可能还需要做一些初始化工作，首页请求首页渲染，最后用户看到界面才可以开始使用。

T1期间包含下面步骤：

- exec

这一步首先fork创建一个新的进程(新建一个进程id)，然后exec来进行进程的替换(给新创建的进程分配独立与父进程的运行所需资源)。
有了这两步，app有了独立的进程id，有了独立的进程空间。执行这个操作是launchd守护进程去做的。

exec是一组系统调用函数，用来替换进程映像，内核通过exec运行我们打开的app进程，将其映射到一个进程空间。

- 加载可执行文件

拷贝可执行文件到内存中，可执行文件中的load commands会被加载。load commands就是来指导dyld进行加载工作的。

ASLR(Address space layout randomization) 用来将进程在随机中的地址进行加载，dyld加载镜像的时候每个段每次的地址都会变化，会和已经编译好的地址存在一定的slides，增加了对某个已知地址攻击的难度。 

- 加载dyld

load commands 中有一条是`LC_LOAD_DYLINKER`，这个时候就会去启动dyld（动态链接器)。

- dyld加载动态库

这一步就是递归的加载可执行文件中用到的动态库。

加载完所有的动态库后，需要一个绑定的步骤，所谓的Fix-ups，rebase和bind。可以去`LC_DYLD_INFO_ONLY` load command去查看rebase和bind的偏移量和偏移大小。

- rebase

指针修正的过程，在可执行文件内部调整指针的指向。

在进行rebase之前，可执行文件只是由内核映射到虚拟内存，还未加载进内存。当rebasing的时候，会从`__DATA`段进行读取，发现没有数据产生page fault，此时内核会将磁盘中将对应的页加载到内存，之后是进行rebasing，可以看到IO操作也是在这一步完成的，这一步会进行改页的签名验证，保证没有被篡改。

因为ASLR，动态库和可执行的虚拟内存地址会加载到随机的地址，所以需要修正`__DATA`中指针，根据实际地址(actual_address)和旧地址(preferred_address)算出slide（actual_address - preferred_address)，将需要rebase的地址加上这个slide。

**这边不是太理解rebase的过程, slide，需要rebase的指针**

- bind

符号绑定的过程，也就是将指针指向可执行外部的地址。

> bind又分bind，weak bind，lazy bind；

因为我们编写的程序中一定会用到系统的库中，所以需要进行符号的地址绑定。

objc中类继承，protcol是non-lazy的，启动就会进行绑定，函数里调用外部函数是lazy的，在第一次调用才会进行绑定。

`__DATA`段中会生成一个`__got`表用来存放外部符号的地址，caller调用的时候就是`__got`表中的一个地址。对于lazy bind的符号，会将其在`__got`表中设置为`__stub_helper` section中的一个地址，第一次调用的时候会调用到`dyld_stub_binder`，这个函数会去到库中进行查找符号将其写入`__got`表中再调用，这样下次就可以直接调用了。

- objc runtime初始化

扫描所有的objc类型，class，protocol，category，然后进行`realizeClass`进行类的初始化（class_rw_t, isa, non-fragile ivars, methods, category)。

- initializers

objc`load`方法的执行；

` __attribute__((constructor))`函数的执行。

---

最后dyld将控制权交给main函数，对应`LC_MAIN` load command。

# dyld 相关

ios16启动优化。

[ref](https://www.emergetools.com/blog/posts/iOS16LaunchTime?utm_source=swiftlee&utm_medium=swiftlee_weekly&utm_campaign=issue_123)

# 启动优化


# References

[https://blog.cnbluebox.com/blog/2017/10/12/dyld2/](https://blog.cnbluebox.com/blog/2017/10/12/dyld2/)

[https://zhuanlan.zhihu.com/p/42951190](https://zhuanlan.zhihu.com/p/42951190)

[http://yulingtianxia.com/blog/2016/10/30/Optimizing-App-Startup-Time/](http://yulingtianxia.com/blog/2016/10/30/Optimizing-App-Startup-Time/)

[https://tech.meituan.com/2018/12/06/waimai-ios-optimizing-startup.html](https://tech.meituan.com/2018/12/06/waimai-ios-optimizing-startup.html)

[https://www.cnblogs.com/qingergege/p/6601807.html](https://www.cnblogs.com/qingergege/p/6601807.html)

[http://www.arkteam.net/?p=2728](http://www.arkteam.net/?p=2728)

[http://www.samirchen.com/ios-app-launching/](http://www.samirchen.com/ios-app-launching/)

[http://lingyuncxb.com/2018/01/30/iOS%E5%90%AF%E5%8A%A8%E4%BC%98%E5%8C%96/](http://lingyuncxb.com/2018/01/30/iOS%E5%90%AF%E5%8A%A8%E4%BC%98%E5%8C%96/)

[http://www.zoomfeng.com/blog/launch-time.html](http://www.zoomfeng.com/blog/launch-time.html)
