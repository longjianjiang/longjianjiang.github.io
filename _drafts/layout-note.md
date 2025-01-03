
# frame

# AutoLayout

## priority

约束有优先级，范围是0-1000，默认是1000。优先满足优先级高的，如果存在两个相同优先级的约束则会冲突；

当某个view的某个约束可能发生变化，可以对该约束进行设置多个，此时需要将优先级进行变化。


```                       
+-------------------------+
|                         |
|    +--------------+     |
|    +--------------+     |
|                         |
|         +-+             |
|         +-+             |
|                         |
+-------------------------+
```

如上图所示，当label的内容为空，btn需要距离父view间距是10。

有两种方案：

1> 保存btn的top约束，设置text的时候，更新约束；
2> btn的top约束设置两个，一个相对于label，一个相对于父view，设置text时，text为空，将label移除，这样才能让优先级低的相对于父view的约束生效。

## contentHuggingPriority & contentCompressionResistancePriority

huge的这个是拒绝放大，也就是缩小；
compress这个是拒绝缩小，也就是放大；

看这样一个Cell：

+------------------------------------------------+
|                                                |
| +-+ +-------------------------------+   +---+  |
| +-+ +-------------------------------+   +---+  |
|                                                |
+------------------------------------------------+

左边一个icon，中间一个标题label，右边一个时长label。约束其实很正常，但是当标题很长的时候，此时约束就冲突了，因为两边两个label扩大的优先级一样，就挡住了时间展示。

解决方法也很简单，将标题label的放大优先级降低，使用`setContentCompressionResistancePriority`即可。

这里需要注意的是，标题label需要设置右边的约束(lessThanOrEqualTo)，否则使用拒绝缩小是不起效果的。

## 问题

今天修一个bug的时候，发现当Cell高度不够展示约束设置的内容的时候，会不展示Label的内容，开始以为约束出问题了。

---

今天遇到一个在11上collectionView一个cell未显示出来的问题，原因是subView设置约束根据同级的subview的subview去设置约束，导致没有显示出来，11以上是正常的，所以设置约束要保障在同一层级；

## Masonry

Masonry 封装了`NSLayoutConstraint`的接口，提供了链式调用的语法。

当view使用xib中设置约束，或者下面提到的快捷方式设置的约束，如果此时去update或者remake是不会起作用的，因为内部存储了通过自身接口创建的约束，不会去更改其他方式创建的约束。

## NSLayoutAnchor

iOS9 Apple提供了一种快捷的方式进行创建约束，如下所示:

{% highlight objc %}
NSLayoutConstraint *constraint = [myView.topAnchor constraintEqualToAnchor:otherView.topAnchor constant:10];
{% endhighlight %}

## 动画

当需要更新约束做动画时，更新完需要后，需要调用父类的`layoutIfNeeded`来使布局立即更新，否则动画没有效果。

因为约束通过算法计算后依然是frame，那么正常情况下约束的更新不会立即去转成frame，所以做动画的时候frame没改变，所以就没有动画效果，而layoutIfNeeded的作用在于让约束的改变立即生效，这样在动画的block中frame改变了，动画也就生效了。

## UIStackView

使用stackView可以少写很多约束，还是比较方便的。stackView可以理解为一个布局的容器，他的大小是由subviews决定的，而且是拥有intrinsicContentSize的。

stackView有水平布局和垂直布局，alignment是布局方向的垂直方向上的对齐方式，distribution是布局方向上的布局策略。

当alignment 和 distribution 都设置为fill的时候，假设是水平方向布局，此时水平方向的左右约束其中一个要设置成模糊，否则和intrinsicContentSize的width约束就冲突了。

---

当UITextView放到stackView中，当变换textView高度约束进行展示内容的时候，有可能出现文字无法显示。

解决方案是按条件将textView的滚动禁止，同时用一个view把textView包起来，这样测试就正常了。

---

当stackView里面有两个label，当其中一个隐藏了此时是不展示padding的，需要注意的是，如果判断有问题，其中一个label设置的文字为nil，此时padding是会展示的。

---

当stackView内部放多个label，而且label的内容可能超过一行的时候，这个时候需要按次序设置后面的label的水平优先级(setContentCompressionResistancePriority)，否则会出现约束的冲突。

---

stackView中放uiview，uiview中放置stackView，这个时候uiview是展示不出来的。

## 模糊约束

+------------------------------------------------+
|  +----------+ +--------------+                 | 
|  |          | |              |                 | 
|  +----------+ +--------------+                 | 
+------------------------------------------------+

如上有两个label，左边是折扣价label，右边是原价label，要保证左边的label一定可以显示的下，右边的label可以被折叠。

首先我们需要设置右边label contentCompressionResistancePriority 为low。但是我们设置了以后会发现左边的label显示的区域比实际文字要长，这个时候需要额外设置右边label距离右边的一个模糊约束，这样左边label的宽度就是内容的宽度了。

```swift
groupBuyPriceLabel.snp.makeConstraints {
    $0.top.equalTo(titleLabel.snp.bottom).offset(6)
    $0.leading.equalTo(titleLabel)
}
originalPriceLabel.setContentCompressionResistancePriority(.defaultLow, for: .horizontal)
originalPriceLabel.snp.makeConstraints {
    $0.top.equalTo(groupBuyPriceLabel).offset(5)
    $0.leading.equalTo(groupBuyPriceLabel.snp.trailing).offset(5)
    $0.trailing.lessThanOrEqualToSuperview().inset(-15)
}
```

## UILayoutGuide

所谓UILayoutGuide是一个虚拟的矩形区域，可以和AutoLayout交互，但是不会加入view的层级也就不会拦截手势相关的消息。

最常见的应该是`safeAreaLayoutGuide`，规定了一个不被导航栏，tabbar，toolbar这些挡住的区域，使用他来设置约束就不用hard code相关的数字了。

`layoutMarginsGuide`就是view的margin；`readableContentGuide`规定了一个相对友好的阅读区域，一般使用在iPad里。

我们也可以自定义一个UILayoutGuide，此时作用就相等于一个容器，子view相对于这个layoutGuide来设置约束。

## intrinsicContentSize 

当自定义view重写了intrinsicContentSize，并且可能intrinsicContentSize可能会根据内容进行改变，这个时候改变之后需要调用invalidateIntrinsicContentSize来进行一次更新。

## layout变化

当某个view根据兄弟view进行设置约束，此时兄弟view进行了一次removeFromSuperview然后重新添加操作后，会导致view的位置进行变化，因为之前的view已经不在了，约束也就找不到了。

## 几个方法

setNeedsLayout，会标记layout需要更新，会在下一次runloop的最后，统一提交一次loop内的所有改动。layoutIfNeeded则是判断当前如果存在需要更新的标记，则立马进行layout的更新，不等到loop结尾。

setNeedsUpdateConstraints，标记约束需要更新，会在下一次runloop调用updateConstraints() / updateViewConstraints()进行更新约束。updateConstraintsIfNeeded()则是判断当前如果存在需要更新的标记，立即调用updateConstraints() / updateViewConstraints()。

正常情况下，更新frame，更新约束，都会触发layout的更新，所以一般情况下，不需要调用上面的这些方法。

一般流程：

```
initWithFrame
| loop start
setNeedsLayout() ,setNeedsUpdateConstraints()
|
update constraints? => updateConstraints()
|
update layout? => layoutSubviews()
|
need display? => drawRect()
| loop end
```

[ref](https://stackoverflow.com/questions/47823639/why-calling-setneedsupdateconstraints-isnt-needed-for-constraint-changes-or-ani)
[ref](https://stackoverflow.com/questions/20609206/setneedslayout-vs-setneedsupdateconstraints-and-layoutifneeded-vs-updateconstra)

# flexbox

flexbox是CSS中使用布局算法。

## container property

- flex-direction

决定了容器内item的摆放方向，有四种选择，row(水平左,默认)，row-reverse(水平右)，column(垂直上)，column-reverse(垂直下)。

- flex-wrap

决定了容器一条轴线上元素放不下，如何换行，有三种选择，nowrap(默认)，wrap(换行第一行在上方)，wrap-reverse(换行第一行在下方)。

- flex-flow

flex-direction 和 flex-wrap 的 简写显示。默认是row, nowrap。

- justify-content

决定了容器里一条轴线上items的对齐方式，有五种选择，flex-start(默认)，center，flex-end，space-between(每个item的间隔相等)，space-around(每个item两侧间隔相等，item之间的间隔是item和边框间隔的2倍)。

- align-items

决定了items在交叉轴上如何对齐。假设交叉轴的方向是从上到下。

有五种选择，flex-start(交叉轴的起点对齐)，flex-end(交叉轴的终点对齐)，center(交叉轴的中点对齐)，baseline(项目第一行文字的基线对齐)，stretch(默认，撑满交叉轴)。

- align-content

## item property

- order

items按order的大小进行排序，默认order是0，order越小排在越前。

- flex-grow

item的放大比例，默认是0，即使空间允许也不放大。

假设四个item，一二四的`flex-grow`是1，三的`flex-grow`是2，那么当存在空余位置，三相比其他占据两倍的位置。

- flex-shrink

item的缩小比例，默认是1，当空间不足，item将缩小。

当某个item`flex-shrink`是0，那么当空间不足的时候，该item不会缩小。

- flex-basis

指定item占据的大小，默认是`auto`，也就是项目原本的大小。

可以指定固定大小，类似`width`, `height`等属性的值。

- flex

flex 是 `flex-grow`, `flex-shrink`, `flex-basis`的简写，默认是`0 1 auto`，后两个可选。

有两个快捷值，auto=> 1 1 atuo, none=> 0 0 auto

- align-self

单独指定item的对齐方式，覆盖`align-items`属性，默认是auto，表示使用容器的`align-items`属性。

# css

在项目的过程中，作为程序员，都希望代码能够复用，提高稳定性。然而现实是残酷的，其他人并不会替你思考这些问题，所以就有很多场景，明明看着一模一样，但是某几个字体就是不一样，大小就是差那么几个像素。那么要处理这种问题，一般有两种方案。

1，继承，基类写基本成员，子类来写布局和属性。这样会导致很多子类，不熟悉的人会很疑惑这些东西都在哪里用的。

2，增加style属性，使用style来重写布局和属性，这样可能会随着类型的增多switch-case也增加。

同样，这里web也给了我们一个很好的思路。内容-样式分离，我们可以做一套类似于css的系统，使用class来设置布局样式，这样布局样式也可以复用了！

# References

[http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)

[https://css-tricks.com/almanac/properties/j/justify-content/](https://css-tricks.com/almanac/properties/j/justify-content/)
