
# frame

# AutoLayout

## priority

约束有优先级，范围是0-1000，默认是1000。当某个view的某个约束可能发生变化，可以对该约束进行设置多个，此时需要将优先级进行变化。

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

## 问题

今天修一个bug的时候，发现当Cell高度不够展示约束设置的内容的时候，会不展示Label的内容，开始以为约束出问题了。

## Masonry

Masonry 封装了`NSLayoutConstraint`的接口，提供了链式调用的语法。

当view使用xib中设置约束，或者下面提到的快捷方式设置的约束，如果此时去update或者remake是不会起作用的，因为内部存储了通过自身接口创建的约束，不会去更改其他方式创建的约束。

## NSLayoutAnchor

iOS9 Apple提供了一种快捷的方式进行创建约束，如下所示:

{% highlight objc %}
NSLayoutConstraint *constraint = [myView.topAnchor constraintEqualToAnchor:otherView.topAnchor constant:10];
{% endhighlight %}

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

# References

[http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)

[https://css-tricks.com/almanac/properties/j/justify-content/](https://css-tricks.com/almanac/properties/j/justify-content/)
