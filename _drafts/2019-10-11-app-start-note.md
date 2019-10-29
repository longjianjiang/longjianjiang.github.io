
app 启动优化

# 启动过程

大致过程如下：

				T1                        T2
点击app图标 -----------> 执行main函数 -----------> didFinishLaunchingWithOptions

T1指的是main函数执行之前，这个时候操作系统将可执行文件加载进内存，执行一些加载&链接工作，最后开始执行main函数。
T2指的是main函数执行之后，到didFinishLaunchingWithOptions执行完毕这一段时间。
T3指的是didFinishLaunchingWithOptions返回后，此时可能还需要做一些初始化工作，首页请求渲染，最后用户看到界面才可以开始使用。

T1期间包含下面步骤：

- exec

这一步首先fork创建一个新的进程，然后exec来进行进程的替换。

exec是一组系统调用函数，用来替换进程映像，内核通过exec运行我们打开的app进程，将其映射到一个进程空间。

- 加载可执行文件

拷贝可执行文件到内存中，可执行文件中的load commands会被加载。

ASLR(Address space layout randomization) 用来将镜像在随机中的地址进行加载，防止溢出漏洞攻击。

- 加载dyld

load commands 中有一条是`LC_LOAD_DYLINKER`，这个时候就会去启动dyld（动态链接器)。

- dyld加载动态库

这一步就是递归的加载可执行文件中用到的动态库。

加载完所有的动态库后，需要一个绑定的步骤，所谓的Fix-ups，rebase和bind。

- rebase

在可执行文件内部调整指针的指向。

- bind

将指针指向可执行外部的地址。

- objc runtime初始化

将分类中的方法插入方法列表；

保证selector的唯一性；

- initializers

objc`load`方法的执行；

# 启动优化


# References

[https://zhuanlan.zhihu.com/p/42951190](https://zhuanlan.zhihu.com/p/42951190)

[http://yulingtianxia.com/blog/2016/10/30/Optimizing-App-Startup-Time/](http://yulingtianxia.com/blog/2016/10/30/Optimizing-App-Startup-Time/)

[https://tech.meituan.com/2018/12/06/waimai-ios-optimizing-startup.html](https://tech.meituan.com/2018/12/06/waimai-ios-optimizing-startup.html)

[https://www.cnblogs.com/qingergege/p/6601807.html](https://www.cnblogs.com/qingergege/p/6601807.html)
