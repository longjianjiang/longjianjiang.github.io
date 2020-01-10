
# 日常开发速查

## rightBarButtonItem 隐藏

```
if (condition) {
    self.navigationItem.rightBarButtonItem.title = @"";
    self.navigationItem.rightBarButtonItem.enabled = NO;
} else {
    self.navigationItem.rightBarButtonItem.title = @"my button title";
    self.navigationItem.rightBarButtonItem.enabled = YES;
}
```

[ref](https://stackoverflow.com/questions/3042818/hide-the-rightbarbuttonitem-of-a-navigation-controller)

## Cell 中存在animationImageView

```
override var isSelected: Bool {
	set {}
	get { return super.isSelected }
}

override var isHighlighted: Bool {
	set {}
	get { return super.isHighlighted }
}
```

[ref](https://stackoverflow.com/questions/27904177/uiimageview-animation-stops-when-user-touches-screen)

## 毛玻璃效果修正

一个播放中心界面，使用的是`.light`的effect，发现很亮，设计稿上就很淡，在effectView的contentView里面加一层半透明(white 0.8)的view后，就会谈很多。

## Cell 圆角

当collectionView背景色为白色，cell背景色是白色，这个时候加了圆角当cell是看不出效果的，因为都是白色。
