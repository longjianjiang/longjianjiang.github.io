
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

## cell中某个view需要做动画

这个时候需要等于listView加载完毕，也就是reloadData()后进行动画。因为UI操作一定是在主线程，所以可以使用GCD将动画操作加到队列中，这样reload完成后，此时view也都加载完毕，可以正常的进行动画显示；

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

# 安全区域相关

默认的listView内容会自动设置contentInset来保证安全区域不显示内容，但是有些UI就是从(0, 0) 开始设计的，所以需要设置如下来消除自动的inset：

```swift
listView.contentInsetAdjustmentBehavior = .never
```

当我们需要知道安全区域的inset的值的时候，通过`safeAreaInsets`来获取。假设要想在listView的cell中获取inset的话，则需要通过superView也就是外面的listView来获取，并且需要在`layoutSubviews`(会被调用多次)时候才去获取，init的时候还没有值。

# UIButton 相关

1> inset

假设有一个btn，包含一段文字和一个图标，文字是会变化的，要求自适应btn的宽度。

首先设置约束不能设置宽度，默认btn会根据内容进行计算出一个`intrinsicContentSize`，但是一般情况下，设计稿中的文字和图片之间以及和btn之间都会存在间距，所以在设置`imageEdgeInset`之前需要先设置`contentEdgeInset`，这样btn内部计算`intrinsicContentSize`的时候就会加上inset，这样接下去设置图片或者文字的inset的时候就可以有位置进行偏移，否则就显示不下了。

例子如下：

```swift
speedButton.contentEdgeInsets = UIEdgeInsets(top: 0, left: 8, bottom: 0, right: 5+6) // 6其实就是下面image需要设置的inset
speedButton.imageEdgeInsets = UIEdgeInsets(top: 0, left: 6, bottom: 0, right: -6) // right需要加-6，否则图片的frame就不对了，需要往右加6，这样图片就不会被挤；iOS13以下不加-6图片显示偏移；
```

# 阴影

layer设置阴影代码如下：

```swift
extension CALayer {
    func applySketchShadow(
        color: UIColor = .black,
        alpha: Float = 0.5,
        x: CGFloat = 0,
        y: CGFloat = 2,
        blur: CGFloat = 4,
        spread: CGFloat = 0) {

        shadowColor = color.cgColor
        shadowOpacity = alpha
        shadowOffset = CGSize(width: x, height: y)
        shadowRadius = blur / 2.0
        if spread == 0 {
            shadowPath = nil
        } else {
            let dx = -spread
            let rect = bounds.insetBy(dx: dx, dy: dx)
            shadowPath = UIBezierPath(rect: rect).cgPath
        }
    }
}
```

如果一个View既要设置圆角也要有阴影，此时可以外面加一个父view设置阴影，父View在添加这个设置了圆角的view。需要注意的是，需要设置父View或者子View的背景色，否则默认透明的阴影显示不出来。

# Timer, WeakProxy

OC中使用YYWeakProxy，swift可以使用如下的方案：

```swift
class TargetProxy {
	private weak var target: CollegeTryMsgTopBannerView?

	init(target: CollegeTryMsgTopBannerView) {
		self.target = target
	}

	@objc func addBarrage() {
		print("add one barrage")
		let node = TextBarrageNode(barrageNode: target?.tryMsg)
		target?.barragePanel?.add(barrage: node)
	}
}
let timer = Timer.scheduledTimer(timeInterval: interval,
							 target: TargetProxy(target: self),
							 selector: #selector(TargetProxy.addBarrage),
							 userInfo: nil,
							 repeats: true)
RunLoop.main.add(timer, forMode: .common)
```

# present

A presentViewController 到B 后，A.presentedViewController就是B，B.presentingViewController就是A

iOS13默认的present不是全屏，所以present之前，需要设置控制器的`modalPresentationStyle`属性，这样才可以全屏的present；

```swift
let shareVC = ColumnPlanShareController()
// 使用 `.fullScreen`会导致不能看到presetning 的控制器
shareVC.modalPresentationStyle = .overCurrentContext
vc?.present(shareVC, animated: true, completion: nil)
```

# URL 中文

```swift
let urlStr = "https://host/path?q=布鲁斯"
let encodeURLStr = urlStr.addingPercentEncoding(withAllowedCharacters: CharacterSet.urlQueryAllowed)
let decodeURLStr = encodeURLStr.removingPercentEncoding
```

# swift protocol property type subclass

[ref](https://stackoverflow.com/questions/32231420/swift-protocol-property-type-subclass)

# 梯形（左上/左下 带圆角）的绘制

绘制圆角的时候角度需要注意，需要从圆心的右边开始为0，然后逆时针方向，分别是90度，180度。

其次就是额外画出交点的圆角，分成两步，首先画线空出半径的宽度，然后在这个空出的半径的矩形中找到远点进行一次`addArc`即可。

```swift
override func draw(_ rect: CGRect) {
	super.draw(rect)

	let diff: CGFloat = 4
	let radius: CGFloat = 2

	let path = UIBezierPath()
	path.lineJoinStyle = .round
	path.move(to: .zero)
	path.addLine(to: CGPoint(x: rect.width-diff, y: 0))
	path.addLine(to: CGPoint(x: rect.width, y: rect.height))

	path.addLine(to: CGPoint(x: radius, y: rect.height))
	path.addArc(withCenter: CGPoint(x: radius, y: rect.height-radius),
				radius: radius,
				startAngle: CGFloat.pi / 2,
				endAngle: CGFloat.pi,
				clockwise: true)

	path.addLine(to: CGPoint(x: 0, y: radius))
	path.addArc(withCenter: CGPoint(x: radius, y: radius),
				radius: radius,
				startAngle: CGFloat.pi,
				endAngle: CGFloat.pi * 1.5,
				clockwise: true)
	path.close()

	let shapeLayer = CAShapeLayer()
	shapeLayer.path = path.cgPath
	shapeLayer.fillColor = UIColor.assistPurpleColor.cgColor

	layer.insertSublayer(shapeLayer, at: 0)
}
```
