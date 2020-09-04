
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

设置view的layer的`cornerRadius`和shadow是可以同时生效的。

如果一个View既要设置圆角也要有阴影，此时可以外面加一个父view设置阴影，父View在添加这个设置了圆角的view。需要注意的是，需要设置父View或者子View的背景色，否则默认透明的阴影显示不出来。

listview 的`clipToBounds`是true，会导致阴影显示不全，此时需要设置为false，才能显示正常；

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

```
modalTransitionStyle = .crossDissolve; // 设置present的渐变效果；
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

# UIImageView ContentMode & clipToBounds

开发中有个页面collectionView，有两组，都是一张网络图片并且知道宽高，第一组距离顶部10；  
因为之前高度直接使用的是375对应的高度，而且contentMode为`xxfill`，默认clipToBounds是false，所以顶部就超出了，之前以为的collectionView未生效。  
设置fill的时候，需要考虑clipToBounds。 

# UserDefaults

UserDefaults 存储字典时，key如果不是string时，此时就不能使用UserDefaults 存储，此时存储的对象不是`Property List`。
UserDefaults 存储Set<Int>时也会失败；

# view 重复添加

比如某些情况需要在cell上额外增加一个提示的view，假设重复很多个提示的view的颜色会变深而且点击事件会无法触发，这个时候就需要适当的做移除操作；

# autoresizing cell

collectionview 当我们遇到高度不固定的，通常会设置约束去自动算高，不过有些内容因为只设置了左右的约束，而宽度是不固定，因为此时cell的宽度还不知道，所以必须通过内部的subview宽度来给cell自己设置宽度；

一种方式是设置一个宽度约束，比如取屏幕宽度减去两边间距，这样有个问题是iPad分屏的时候会出问题；

另一种方式是，在flowLayout的delegate里使用collectionView的宽度返回，然后在cell中的prefer修改layoutAttribute方法中使用宽度去根据autolayout算实际的高度，然后更新layoutAttribute；

第三种情况是，cell的大小是固定的，因为使用了estimatedSize同时约束是根据cell的大小来设置的，这个时候同样会根据约束去计算出实际的大小，但是还是有可能宽度不是collectionView的宽度，这个时候直接使用flowLayout代理的设置的itemSize，直接在cell的prefer方法中返回方法中的layoutAttribute中即可（此时layoutAttribute就是flowLayout中的）；

---

笔者今天在写一个页面的时候，有三个section，第一个section高度固定，第二个section是一组搜索词，高度固定，宽度通过cell的prefer的方法中计算；第三个section是固定高度的cell；在iOS11上的设备显示的时候，第三个section的cell和header会往上偏移，猜想是第二组的section高度还没计算出来，第三个section就展示出来了，所以导致了重叠，解决方法就是第三个section的header转cell，cell也实现prefer方法，手动赋值高度，同时flowLayoutDelegate itemSize的高度返回1，这样就可以正常显示了。

# 手势

今天遇到一个类似抖音的视频页面，视频View有添加pan手势处理快进，然后和scrollView的pan手势冲突了。

解决方法，实现视频控制view手势的`shouldRecognizeSimultaneouslyWith`代理方法，尝试返回true即可同时识别两个手势，这样scrollView的pan手势就可以生效了。

---

今天在一个模块里，collectionView的宽度固定显示三个，cell个数则可能不满3个，此时也需要记录点击事件，所以需要添加一个手势，需要实现`shouldReceiveTouch`方法，判断点击的点是否存在对应的cell；

```swift
func gestureRecognizer(_ gestureRecognizer: UIGestureRecognizer, shouldReceive touch: UITouch) -> Bool {
	if goodsListView?.indexPathForItem(at: touch.location(in: goodsListView)) != nil {
		return false
	}
	return true
}
```

# lottie

一个按钮左边👍图片，右边数字，点击的时候放在图片上面播放一次lottie动画，开始播放会透出未选中图片的灰色。

查看了下是因为lottie的那个view背景是透明的，所以将lottieview 背景色设置为白色即可。

# collectionView 刷新

listView中某个header有选中状态，去更新后发现没有生效，因为更新后触发了reload操作，所以header又重置了，所以没有看到刷新效果。

# CAGradientLayer

CAGradientLayer 的startPoint 和 endPoint 有时间直接看蓝湖上面的代码会不准确。
两点之间的方向会影响效果，有水平方向和垂直方向，或者对角线方向。

# NumberFormatter

当需要将数字改成多少万的时候，可以使用numberFormatter来做。如下是一个例子：

```swift
static func getReadableCnt(_ cnt: Int, addedSpacing: Bool = false) -> String {
	let count = Double(cnt) / 10000.0
	let formatter = NumberFormatter()
	formatter.minimumFractionDigits = 0 // 1.0万显示成1
	formatter.maximumFractionDigits = 1 // 保留一位小数

	if count >= 1 {
		let prefix = formatter.string(from: NSNumber(value: count)) ?? String(format: "%.1f", count)
		if addedSpacing {
			return String(format: "\(prefix) 万", count)
		}
		let countStr = String(format: "\(prefix)万", count)
		return countStr
	} else {
		if addedSpacing {
			return "\(cnt) "
		}
		return "\(cnt)"
	}
}
```

[ref](https://nshipster.com/formatter/)

# UICollectionViewLayout

在写一个瀑布流的layout的时候，开始可以正常显示，每次reloadData后，cell的显示就不对了。开始以为是layout计算出问题了，后来发现其实问题是图片的约束设置所导致的。

```swift
imgView.snp.remakeConstraints {
	$0.top.leading.trailing.equalToSuperview()
	$0.height.equalTo(imgView.snp.width).multipliedBy(item.coverUrl.imgPercent)
}
```

开始高度约束是基于imgView的父view的宽度来设置的，每次reloadData就会出现高度拉伸的问题，后来改成基于自己的宽度就正常了。
