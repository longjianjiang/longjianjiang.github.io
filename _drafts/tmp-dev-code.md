
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
