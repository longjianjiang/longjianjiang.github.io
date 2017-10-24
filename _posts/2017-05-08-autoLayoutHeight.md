---
layout: post
title:  "认识Auto Layout"
date:   2017-05-08
excerpt: "开发中我们经常会遇到算高问题，比如文本控件中显示的内容是不确定的，再比如cell中的内容是不确定的，这样的情况都会导致该view的高度是不确定的，这个时候不可避免的就需要进行计算高度。"
tag:
- iOS
comments: true
---

> Auto Layout是用来进行界面布局的一种方式，一种以约束的方式进行布局`View`。

### 前言

开发中我们经常会遇到`算高`问题，比如文本控件中显示的内容是不确定的，再比如`cell`中的内容是不确定的，这样的情况都会导致该`view`的高度是不确定的，这个时候不可避免的就需要进行计算高度。

#### intrinsicContentSize

下面来看个栗子：`view`的层级如下图所示：

![屏幕快照 2017-05-05 18.08.27.png](http://ocigwe4cv.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-05%2018.08.27.png)

代码如下:

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self.view addSubview:self.contentView]; // 橙色
    [self.contentView addSubview:self.msgLabel];
    [self.contentView addSubview:self.avatarView];
    
    [self.contentView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(self.view).offset(20);
        make.left.right.equalTo(self.view);
    }];
    
    [self.msgLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.contentView).offset(5);
        make.top.equalTo(self.contentView);
        make.right.lessThanOrEqualTo(self.contentView).offset(-5);
    }];
    [self.avatarView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(self.msgLabel.mas_bottom).offset(10);
        make.left.equalTo(self.contentView).offset(5);
        make.bottom.equalTo(self.contentView).offset(-10);
        make.width.mas_equalTo(250);
        make.height.mas_equalTo(250);
    }];
}
```

上述代码在设置约束的时候第一没有设置`msgLabel`的高度，第二也没有设置`contentView`的高度，但是却可以正常的显示。这很好的说明了这些Auto Layout都已经帮我们做好了，也就是说我们并不需要像之前使用`boundingRect`方法来计算文本内容的高度。
Auto Layout内部是如何计算出`msgLabel` 、`contentView`的高度来的呢？`intrinsicContentSize`功不可没😄。

`intrinsicContentSize`是`UIView`的一个属性，`intrinsicContentSize`是基于`view`的内容进行计算出来的，比如说`UILabel`的`intrinsicContentSize`是基于`text`和`font`(猜想内部可能通过调用`boundingRect`方法来计算出`size`)。但是并不是所有的view都有`intrinsicContentSize`，常见的`view`的`intrinsicContentSize`如下图所示：

![屏幕快照 2017-05-07 09.41.27.png](http://ocigwe4cv.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-07%2009.41.27.png)

如上例，我们在设置约束的时候，可以使用`view`的`intrinsicContentSize`在我们的布局中，这样可以使我们的布局适应当view的内容发生改变，同时也可以减少设置约束的数量。

但是如果我们使用`intrinsicContentSize`在我们的布局中，就会引入一个新的问题，有时我们需要额外的设置view的 `content-hugging` 、`compression-resistance`优先级。

Auto Layout 在设置view的`intrinsicContentSize`的时候，在水平和垂直方向分别设置了一对约束。
``` 
// Compression Resistance
View.height >= 0.0 * NotAnAttribute + IntrinsicHeight
View.width >= 0.0 * NotAnAttribute + IntrinsicWidth
 
// Content Hugging
View.height <= 0.0 * NotAnAttribute + IntrinsicHeight
View.width <= 0.0 * NotAnAttribute + IntrinsicWidth

```
同时`content-hugging`、`compression-resistance`约束都有各自的优先级，`compression-resistance`优先级默认为750，`content-hugging`优先级默认为250。因此对于一个`view`来说我们更容易将其扩大，比如说我们可以将一个按钮的尺寸变大，但是如果将其变小就可能会使按钮内部文字显示不全。

系统提供了两个方法，通过更改`content-hugging`、`compression-resistance`的优先级，可以更改view中subViews中那个subView应该被拉伸或者缩小。

```
- (UILayoutPriority)contentHuggingPriorityForAxis:(UILayoutConstraintAxis)axis NS_AVAILABLE_IOS(6_0);
- (void)setContentHuggingPriority:(UILayoutPriority)priority forAxis:(UILayoutConstraintAxis)axis NS_AVAILABLE_IOS(6_0);

- (UILayoutPriority)contentCompressionResistancePriorityForAxis:(UILayoutConstraintAxis)axis NS_AVAILABLE_IOS(6_0);
- (void)setContentCompressionResistancePriority:(UILayoutPriority)priority forAxis:(UILayoutConstraintAxis)axis NS_AVAILABLE_IOS(6_0);
```

> PS: 笔者对于这两个优先级的使用中，发现同样的界面同样的约束，在IB中，系统会自动更改上述两个约束的优先级，但是手写代码中没有更改的情况下，IB中显示效果和手写代码的显示效果是一样的。

#### Fitting Size

之前的IntrinsicContentSize是作为约束的一部分（帮我们生成高度和宽度的约束）；而FittingSize则是作为Auto Layout根据所设置的约束布局完成生成view的size结果（注意这要求我们所设置的约束必须是完整同时不能有冲突）。

常见的应用就是Cell中根据内容自动计算高度，用到的就是下面的方法。
```
- (CGSize)systemLayoutSizeFittingSize:(CGSize)targetSize NS_AVAILABLE_IOS(6_0);
```

下面同样看一个栗子，view的层级图如下所示（PS：内容摘自sunny UITableView-FDTemplateLayoutCell Demo中的data.json）：

![屏幕快照 2017-05-08 15.31.25.png](http://ocigwe4cv.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-08%2015.31.25.png)

Cell中有四部分内容：`titleLabel`,` contentLabel`, `contentImageView`,` usernameLabel`, 同时`titleLabel`,` contentLabel`, `contentImageView`都是可能没有的。 所以tableview在返回高度的时候必然要根据内容显示计算cell的高度。但是这时我们可以利用Auto Layout的fitting size，帮我们计算cell内容的高度，下面就是返回tableViewCell高度方法的实现：

```
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
//    return [tableView fd_heightForCellWithIdentifier:@"cell" cacheByIndexPath:indexPath configuration:^(FDCustomCell* cell) {
//        [self configureCell:cell atIndexPath:indexPath];
//    }];
    
    static FDCustomCell *cell = nil;
    static dispatch_once_t onceToken;
    
    dispatch_once(&onceToken, ^{
        cell = [self.tableView dequeueReusableCellWithIdentifier:@"cell"];
    });

    [self configureCell:cell atIndexPath:indexPath];
    CGFloat height = [cell systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;
    return height;
}
```


#### sizeThatFits：

`sizeThatFits:`和之前的`systemLayoutSizeFittingSize:`效果相似，根据给定的一个size（笔者认为该参数相当于一个约束，通过调用`boundingRect`方法计算文本的高度）。

下面来看一个栗子，view的层级图如下所示

![屏幕快照 2017-05-08 16.37.58.png](http://ocigwe4cv.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-05-08%2016.37.58.png)

如图所示红色的是`label`，白色的为`textView`。之前在说到`IntrinsicContentSize`的时候，提到UITextView在可以滚动的时候是没有`IntrinsicContentSize`的，此时可以使用`sizeThatFits：`方法来计算出`textView`内容的高度，进行更新约束，这样就可以显示，否则图中白色的`textView`是不能显示出来的，使用`sizeThatFits：`方法的代码如下所示：

```
- (void)viewWillLayoutSubviews {
    [super viewWillLayoutSubviews];
    CGSize size = [self.textView sizeThatFits:CGSizeMake(self.view.frame.size.width - 20, 0)];
    [self.textView mas_updateConstraints:^(MASConstraintMaker *make) {
        make.width.mas_equalTo(size.width);
        make.height.mas_equalTo(size.height);
    }];
}
```


### 总结
根据以上，我们知道在以后的开发中合理使用Auto Layout，之前那些算高度的代码，都可以让Auto Layout完成。欢迎各位指出本文的错误！
最后本文中的例子的完整代码点击[下载](https://github.com/longjianjiang/BlogDemo)
