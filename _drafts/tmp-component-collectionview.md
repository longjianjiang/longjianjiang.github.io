
# ListView 组件化

一般情况下首页数据复杂的情况下，ListView中存在多个section，按照传统的方式，controller不可避免的出现很多if/else。此时可以做一些中转，将这部分if/else操作转移到另一个对象。

不过一种更优的选择则是将每个section自治，这样可以做到更大粒度的复用。

业界的IGListKit就是做这个事情的，通过adapter拆分出多个sectionController，每个sectionController管理内部的cell。

# References

[https://xiangwangfeng.com/2019/07/20/UITableView-%E7%BB%84%E4%BB%B6%E5%8C%96/](https://xiangwangfeng.com/2019/07/20/UITableView-%E7%BB%84%E4%BB%B6%E5%8C%96/)

[https://www.jianshu.com/p/f0a74d5744b8](https://www.jianshu.com/p/f0a74d5744b8)

[https://triplecc.github.io/2017/06/23/2017-06-23-ru-he-kuai-su-da-jian-biao-dan-jie-mian/](https://triplecc.github.io/2017/06/23/2017-06-23-ru-he-kuai-su-da-jian-biao-dan-jie-mian/)

[http://zxfcumtcs.github.io/2018/09/17/ListSceneTemplating/](http://zxfcumtcs.github.io/2018/09/17/ListSceneTemplating/)
