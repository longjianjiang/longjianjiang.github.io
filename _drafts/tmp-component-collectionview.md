
# ListView 组件化

一般情况下首页数据复杂的情况下，ListView中存在多个section，按照传统的方式，controller不可避免的出现很多if/else。此时可以做一些中转，将这部分if/else操作转移到另一个对象。

不过一种更优的选择则是将每个section自治，这样可以做到更大粒度的复用。

业界的IGListKit就是做这个事情的，通过adapter拆分出多个sectionController，每个sectionController管理内部的cell。

现在项目如果某个页面分组比较多，目前会采取一种先定义一个`Section`的协议，里面有关于CollectionView delegate和datasource相关的方法，并对其中的一些提供默认实现，如下所示:

{% highlight swift%}
protocol XXXSection: class {
    func numberOfItems() -> Int

    func sectionHeaderSize(for collectionView: UICollectionView) -> CGSize
    func sectionFooterSize(for collectionView: UICollectionView) -> CGSize

    func cellSize(at indexPath: IndexPath, in collectionView: UICollectionView) -> CGSize
    func configCell(for collectionView: UICollectionView, at indexPath: IndexPath) -> UICollectionViewCell

    func sectionInset() -> UIEdgeInsets
    func minimumLineSpacing() -> CGFloat
    func minimumInteritemSpacing() -> CGFloat

    func configSectionHeader(for collectionView: UICollectionView,
                             at indexPath: IndexPath) -> UICollectionReusableView

    func configSectionFooter(for collectionView: UICollectionView,
                                at indexPath: IndexPath) -> UICollectionReusableView

    func didSelected(indexPath: IndexPath, in collectionView: UICollectionView)
}
{% endhighlight %}

之后每一组section创建并遵守这个协议，实现基本的dataSource的方法，cell个数，cell大小，cell创建。这个时候控制器的代码就大大减少，拿到sections取其中的section的实现即可。

还有一个问题就是，section中可能存在某些操作需要控制器进行中转，现在的做法是定义了一组Action，通过subject进行VM和section之间的绑定，控制器监听VM的subject处理各个事件。

这样基本实现的组件化的ListView，但是缺点是Section的协议没有进行统一的抽象，导致每次需要重复创建。

---


# References

[https://xiangwangfeng.com/2019/07/20/UITableView-%E7%BB%84%E4%BB%B6%E5%8C%96/](https://xiangwangfeng.com/2019/07/20/UITableView-%E7%BB%84%E4%BB%B6%E5%8C%96/)

[https://www.jianshu.com/p/f0a74d5744b8](https://www.jianshu.com/p/f0a74d5744b8)

[https://triplecc.github.io/2017/06/23/2017-06-23-ru-he-kuai-su-da-jian-biao-dan-jie-mian/](https://triplecc.github.io/2017/06/23/2017-06-23-ru-he-kuai-su-da-jian-biao-dan-jie-mian/)

[http://zxfcumtcs.github.io/2018/09/17/ListSceneTemplating/](http://zxfcumtcs.github.io/2018/09/17/ListSceneTemplating/)
