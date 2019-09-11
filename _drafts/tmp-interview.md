# 1

最近面试一般问得问题：mrc arc block 循环引用 进程线程 tcp udp runloop runtime 分类 代理 weak assign http https socket也问过 但是不会 音视频 直播 性能优化 你最拿手的是什么 工作中遇到的问题 封装过啥 内存泄露咋处理 代码约束咋做的 生命周期 消息撤回怎么实现的 数据持久化 sql语句 百万级数据查询插入怎么优化 kvc kvo使用场景 copy关键字 gcd说说 事件响应链 文件下载上传怎么做的 图片选择怎么做的 arc有什么缺陷 kvc kvo不能用了怎么办 客户端缓存机制 gcd加锁 automic nonautomic git svn h5交互 地图定位怎么实现的 怎么绑定库 coredata用过没 平常怎么测试的 数据库优化 线程的种类和使用场景

# 2

[ref](http://hl1987.com/2017/09/05/%E8%85%BE%E8%AE%AFSNG%E6%9F%90%E9%83%A8%E9%97%A8iOS%E9%9D%A2%E8%AF%95%E7%BB%8F%E5%8E%86/)

# 3

[ref](https://juejin.im/post/5b5a8e7e518825615e6f6c11)

# 4

[ref](https://github.com/lzyy/iOS-Developer-Interview-Questions)

# 5

[ref](http://www.zoomfeng.com/blog/ios-level-up.html)

# 6

电话面了两次，第一次电面内容如下：
1，MRC和ARC的区别
2，autorelease与release区别
3，自动释放池什么时候被释放
4，weak怎么实现的
5，在oc里对struct进行copy会发生什么，比如对CGPoint 进行copy..
6，如果自己写一个srollView要注意哪些？特别是在内存管理方面，怎么实现类似UTTableView的重用机制
7，看你简历上写着熟练c++, 那c++ virtual怎么实现的？
8，怎么实现静态多态？
9，oc中frame bounds什么区别。。然后什么时候大小会不一样（提示：旋转的时候）
10，有没有考虑过nsnotification发出的通知相关的线程安全？
第二次电面内容如下：
1，介绍项目，哪些技术， 哪些难点，怎么克服？
2，TableView dataSource, delegate分别有哪些方法？
3，有阅读一些优秀的源码没？答：SDWebImage....接着问了SDWebImage的实现细节，比如如果有10个cell需要下载图片，那有几个线程在同时下载？SDWebImage开了几个线程？
4，平时怎么查找bug，解决bug的？
5，看我简历里的项目有涉及UIWebView的，问我UIWebView有没有去了解性能问题
6，CALayer和UIView透明度设置参数分别是？
第三次要求去北京现场面试，开场自我介绍，讲到了自己的毕设，然后开始问我的毕设相关知识点，紧着着还是介绍印象最深刻的项目，让画出项目架构图，聊了聊我做过的项目，最后问我有什么问题要问他。
