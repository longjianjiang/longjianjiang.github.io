
# 日常开发速查

```
出现问题，下断点调试，view debug查看层级，而不是反复的重试或者查看代码，光看是看不出问题的。
```

## navigationBar

有时候改动导航栏，为了不影响其他页面，需要进行disappear重置。

需要注意的时候，需要在willDis的时候进行操作，didDis的时候，此时已经出栈，取self.navigationController为空。

## navigationItem

### rightBarButtonItem 隐藏

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

### titleView

有的时候需要展示自定义的titleView，想要撑满，需要修改一下自定义view的intrinsicContentSize，如下所示:

```swift
override var intrinsicContentSize: CGSize {
	return UIView.layoutFittingExpandedSize
}
```

## cell中某个view需要做动画

这个时候需要等于listView加载完毕，也就是reloadData()后进行动画。因为UI操作一定是在主线程，所以可以使用GCD将动画操作加到队列中，这样reload完成后，此时view也都加载完毕，可以正常的进行动画显示；

listView reloadData默认没有提供完成的回调，可以通过下面的方式进行设置完成的回调。因为在performBatchUpdate和reloadData混合使用的时候，一定得保证reloadData执行完成后，才能performBatchUpdate，否则会触发数据源不一致的异常；

{% highlight oc%}
[CATransaction begin];
[CATransaction setCompletionBlock:^{
	// add some code
}];
[self.msgTableView reloadData];
[CATransaction commit];
{% endhighlight %}

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

上述方式如果在tableview中咩有效果，解决方法可以将cell selection效果设置为none，通过添加手势事件对外抛出处理；

## 毛玻璃效果修正

一个播放中心界面，使用的是`.light`的effect，发现很亮，设计稿上就很淡，在effectView的contentView里面加一层半透明(white 0.8)的view后，就会谈很多。

---

effectView 做动画，模糊和透明之间改变。

```swift
// 首先去掉 UIVisualEffectView 原来的 effect。
self.effectView.effect = nil


// 现在动画添加一个新的 effect。
let blurEffect = UIBlurEffect(style: .Light)

UIView.animateWithDuration(1.5, delay: 0, usingSpringWithDamping: 1.0, initialSpringVelocity: 0, options: .CurveEaseOut, animations: {
    self.effectView.effect = blurEffect
}) { _ in }
```

[ref](https://www.jianshu.com/p/97597719f0fa)

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

上面的例子(同时设置了图片和文字，并且宽度是自适应的)在设置粗体显示的手机上，显示的内容就会有重叠，导致显示不完全出现...这样的效果。
其中一个解决方法如下：

```swift
btn.titleLabel?.lineBreakMode = .byClipping
```

[ref](https://zhoujinying.github.io/2019/09/12/UIButton-%E5%9C%A8%E7%B2%97%E4%BD%93%E6%96%87%E6%9C%AC%E4%B8%8B%E7%9A%84bug/)

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

类型购物车底部的view的上方会加阴影，此时需要将上方的listView离bottomView有一些间距，不然加了阴影被盖住了，就看不出了。
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

---

Timer里面有两种初始化的方法，timerWithTimeInterval这种创建的timer会立马fire，scheduledTimerWithTimeInterval这种创建的timer会等到给定时间后fire。

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

---

点击发送按钮，present一个发送页面，发送页面按钮点击后需要先`dismiss`当前发送页面，animated设置为false，在completion里面进行present新的页面。

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

---

今天加载一张网络图，发现没有显示处理，然后在浏览器打开了图片链接看了下，看到底部有一条内容，开始以为是图片加载有问题，后来发现这张图顶部90%是透明，只有底部10%有一段信息，而imageView设置了fill，所以其实显示的是中间透明的部分，显示效果就是默认的背景色。

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

---

某个view中同时添加pan和tap手势，需要实现shouldRequireFailureOf方法，否则两个会产生冲突：

```swift
func gestureRecognizer(_ gestureRecognizer: UIGestureRecognizer,
					   shouldRequireFailureOf otherGestureRecognizer: UIGestureRecognizer) -> Bool {
	if gestureRecognizer.isKind(of: UITapGestureRecognizer.self)
		&& otherGestureRecognizer.isKind(of: UIPanGestureRecognizer.self) {
		return true
	}
	return false
}
```

## 判断手势的移动方向

使用手势获取速度`velocity(in view: UIView?)`，判断两个方向正负，例子如下：

```swift
extension UIPanGestureRecognizer {

    public struct PanGestureDirection: OptionSet {
        public let rawValue: UInt8

        public init(rawValue: UInt8) {
            self.rawValue = rawValue
        }

        static let Up = PanGestureDirection(rawValue: 1 << 0)
        static let Down = PanGestureDirection(rawValue: 1 << 1)
        static let Left = PanGestureDirection(rawValue: 1 << 2)
        static let Right = PanGestureDirection(rawValue: 1 << 3)
    }

    private func getDirectionBy(velocity: CGFloat,
                                greater: PanGestureDirection,
                                lower: PanGestureDirection) -> PanGestureDirection {
        if velocity == 0 {
            return []
        }
        return velocity > 0 ? greater : lower
    }

    public func direction(in view: UIView) -> PanGestureDirection {
        let velocity = self.velocity(in: view)
        let yDirection = getDirectionBy(velocity: velocity.y, greater: PanGestureDirection.Down, lower: PanGestureDirection.Up)
        let xDirection = getDirectionBy(velocity: velocity.x, greater: PanGestureDirection.Right, lower: PanGestureDirection.Left)
        return xDirection.union(yDirection)
    }
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

---

当设置了estimatedSize后，更新某一组数据的时候，reloadData或者reloadSection都会导致collectionView滚动到顶部。

此时需要调整的地方一是每个itemSize返回实际的高度，二是estimatedSize的高度设置单个cell可能最大的高度。

---

如果collectionView需要展示header，viewForSupplementaryElementOfKind方法不能返回[UICollectionReusableView new]，会导致异常说没有使用dequeue方法创建。

同时viewForSupplementaryElementOfKind的调用是在referenceSizeForHeaderInSection之后，当返回当header尺寸不为0，才会触发。

# UIScrollView

设置scrollView只能水平方向上滚动的一种方式，设置delegate，didScroll修改y。

```swift
func scrollViewDidScroll(_ scrollView: UIScrollView) {
	if scrollView.contentOffset.y > 0 || scrollView.contentOffset.y < 0 {
	   scrollView.contentOffset.y = 0
	}
}
```

当需要监听scrollView移动方向和停止滚动时，快停止滚动时会触发停止减速的回调，但是之后还有继续触发一次didScroll的回调，所以此时需要在didScroll的回调判断滚动的距离，将过小的值过滤，来达到触发滚动停止的执行。

```swift
func subscribeScroll() {
	var oldY: CGFloat = 0
	scholarshipScrollView.rx.contentOffset
		.throttle(RxTimeInterval.milliseconds(100), scheduler: MainScheduler.instance)
		.distinctUntilChanged()
		.subscribe(onNext: { [weak self] point in
			let newY = point.y

			let diff = abs(newY-oldY)
			var isHide = newY > oldY && diff > 2
			if newY <= 0 { isHide = false }

			self?.flipView.updateLeftViewDisplay(hide: isHide)
			oldY = newY
	}).disposed(by: rx.disposeBag)

	scholarshipScrollView.rx.didEndDecelerating.skip(1).subscribe(onNext: { [weak self] _ in
		self?.flipView.updateLeftViewDisplay(hide: false)
	}).disposed(by: rx.disposeBag)
}
```

---

实现类似系统短信，左滑出现时间的效果。可以在tableview的外面放置一层scrollView，因为scrollView可以支持上下左右两个方向的滑动，此时需要保持scrollView和tableview的frame是一样的。

```swift
scrollView = UIScrollView()
scrollView.userInteractionEnalble = false
scrollView.alwaysBouncingHorizontal = true

tableview.addGestureRecognizer(scrollView.panGestureRecognizer)
```

在didScroll的代理方法，分别将 x、y 应用到响应的属性即可，y 轴就给 TableView，让其可以上下卷动，x 轴就给每个 cell，让其做自己的处理。

[ref](https://www.jianshu.com/p/5804fa72aaed)

# UITableView

## tableViewFooter 

设置footer的时候，需要指定frame，指定bounds会出现偏移的情况；

## 滚动到顶部

首先尝试`scrollToRow`，有些情况，使用ScrollView的方法可能未完全移动到顶部；

## scrollsToTop

当ScrollView嵌套会导致点击状态栏回到顶部失效，解决方案只保持一个ScrollView的scrollsToTop为true，其他为false，这样系统就知道滚动哪一个了。

## estimatedRowHeight

今天在一个设置了自动算高的tableview里面，一共4页数据，滚动到底部，此时往上滚动的生活，会不断的抖动很难滚动到顶部。

当使用了自动的高度，此时tableview会推迟计算contentSize，当滑动的时候才会去计算每个cell的高度，所以从底部往上滑动一直滑不上去的原因是因为contentSize这个时候是不对的。

解决方案是存储每个cell的高度，[代码](https://stackoverflow.com/questions/28244475/reloaddata-of-uitableview-with-dynamic-cell-heights-causes-jumpy-scrolling)如下：

```swift
var cellHeights = [IndexPath: CGFloat]()

func tableView(_ tableView: UITableView, willDisplay cell: UITableViewCell, forRowAt indexPath: IndexPath) {
    cellHeights[indexPath] = cell.frame.size.height
}

func tableView(_ tableView: UITableView, estimatedHeightForRowAt indexPath: IndexPath) -> CGFloat {
    return cellHeights[indexPath] ?? UITableView.automaticDimension
}
```

[ref](https://kangzubin.com/uitableview-estimatedrowheight/)

## rowForHeight返回0，还展示

列表里面有两个section，第一个section数据为空时，cell高度返回0。发现cell还是会展示出来，这个时候需要设置 cell.clipToBounds = YES。这样就不会展示了。

## plain 和 group 区别

plain的时候，设置section header默认会有吸顶效果。

如果不需要吸顶效果，则需要实现section footer方法，返回高度为0.01，不能为0，当为0时系统会取默认值。还需要设置tableviewHeader为一个0.01的view。

[ref](https://honkersk.github.io/2018/09/15/UITableViewStylePlain%E5%92%8CUITableViewStyleGrouped%E6%A0%B7%E5%BC%8F%E7%9A%84%E5%8C%BA%E5%88%AB%E5%8F%8A%E6%95%88%E6%9E%9C%E5%A4%84%E7%90%86/)

# UICollectionView

## contentInset

今天遇到一个问题，collectionView水平方向设置了contentInset，此时默认flowLayout的size代理方法宽度返回了实际的宽度，导致layout内部一直在添加row，表现是界面卡住，内存一直在增长，直到crash。

---

最近首页的一个需要是要将collectionView下拉更新的位置放在第一个section下面。

所以需要设置collectionView的contentInset，同时再section的横滑的view放到控制器的view上，这样下拉的时候，seciton可以固定。

此时需要处理的是，监听collectionView的滚动，然后改变section的位置，代码如下：

```swift
func updateWikiListView(_ offset: CGFloat) {
	if offset < -pagingViewTopInset {
		frame.origin.y = view.safeAreaInsets.top
		return
	}

	var scrollY: CGFloat = 0
	if offset < 0 {
		scrollY = offset + pagingViewTopInset
	} else if offset == 0 {
		scrollY = pagingViewTopInset
	} else {
		scrollY = pagingViewTopInset + offset
	}
	if scrollY < 0 { scrollY = 0 }
	
	frame = CGRect(x: 0, y: view.safeAreaInsets.top - scrollY,
								width: view.width, height: homeWikiListHeight)
}
```

---

需要加一个延时，重新设置inset才会生效。

## willDisplay 方法

今天在一个礼物页面，类似抖音直播送礼物，选中某个礼物需要做选中的UI展示。开始UI展示放在了cellForRow方法，这个时候的问题是，当滑动到下一页的时候，再回到上一页，选中状态没有展示，打断点发现其实走到了处理选中UI的方法中。

后面本地测试，发现加延时可以生效，所以想到把选中UI更新的逻辑放在willDisplay方法，试了下，就正常了。在文档中也发现，有这样的建议：

Use the collectionView:willDisplayCell:forItemAtIndexPath: delegate method to make any changes to the appearance of the cell to reflect its visual state such as selection.

# UISearchController

今天遇到一个问题，连续push两个UISearchContainerController，第二个搜索页面的输入框无法编辑。解决方案是在自定义的UISearchContainerController中viewWillAppear设置`definesPresentationContext`为true，viewWillDisappear设置`definesPresentationContext`为false。

[ref](https://www.jianshu.com/p/b065413cbf57)

# UINavigationBar

做一个个人主页的页面，有一张背景图会置顶展示，此时导航栏透明，滚动导航栏展示出来。

这个时候，放置背景图的header中imgView超出`safeAreaGuide`顶部的高度(大于88)。需要设置tableview的`clipToBounds`为false。

这样tablevie可以保持原有位置，导航栏也一直存在，需要做的就是滚动的时候监听，改变导航栏的是否透明和style即可，比较方便。

# UILabel

当UILabel设置了富文本，用到了`paragraphStyle`，这个时候记得设置段落的`lineBreakMode`，默认并不是`byTruncatingTail`，所以会出现文本超过不显示`....`。

# WebView

加载富文本，可能会出现大小不对的情况，需要将返回的富文本加上html head进行包一下。

```swift
func getHTMLString(_ bodyHTML: String) -> String {
	let head = "<head>"
				+ "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0, user-scalable=no\"> "
				+ "<style>img{max-width: 100%; width:100%; height:auto;}*{margin:0px;}</style>"
				+ "</head>"

	return "<html>" + head + "<body>" + bodyHTML + "</body></html>"
}
```

# String

```swift
let spaceAndNewLineCount = str.reduce(0) { $1.isWhitespace || $1.isNewline ? $0 + 1 : $0 } ?? 0
let trimmed = str.filter { !" \n\t\r".contains($0) }
// 这个系统的方法，只会过滤两边的空格比如 " xxx xxx "会变成"xxx xxx"，中间的空格不会过滤。
// 当是" xxx xxxx  d",当这样最后空格有新的字符，此时过滤的结果还是原字符串。
// 和想象中名字的效果差了不少。
let trimStr = text?.trimmingCharacters(in: .whitespacesAndNewlines)
```

# Number

```swift
向上取整：float ceilf(float); double ceil(double);

向下取整：float floorf(float); double floor(double);

四舍五入：float roundf(float); double round(double);
```

# Xib

发现在xib中创建一个custom的button，button只有一张图片，展示效果并不居中，换成代码的方式来创建就正常了。

# UITabbar

当需要点击tabbaritem的时候，做动画，可以实现tabbarVC的代理，在selected方法里面做取tabbar的subview，然后找到imgView，给imgView加动画。

```swift
extension UITabBarController {
    func animationItem() {
        var itemViews = tabBar.subviews.filter { view -> Bool in
            return "\(view.classForCoder)" == "UITabBarButton"
        }
		// 需要排序，顺序可能是不对的，不然根据selectedIndex取的item不对。
        itemViews.sort(by: { $0.frame.minX < $1.frame.minX })

        guard itemViews.count > selectedIndex else {
            return
        }

        // 当有的时候某个tabitem显示返回顶部之类的文字，切换时需要设置为默认的标题；
		if selectedIndex != 0, let label = itemViews.first?.subviews.last as? UILabel {
			label.text = "首页"
			label.textAlignment = .center
		}

        let itemView = itemViews[selectedIndex]
        guard let itemImageView = itemView.subviews.first as? UIImageView else {
            return
        }

        let impliesAnimation = CAKeyframeAnimation(keyPath: "transform.scale")
        impliesAnimation.values = [1.0, 0.8, 1.0]
        impliesAnimation.duration = 0.2
        impliesAnimation.calculationMode = CAAnimationCalculationMode.cubic
        itemImageView.layer.add(impliesAnimation, forKey: nil)
    }
}

```

# UIView

当父view设置isHidden后，此时会隐藏整个view，包括子view，但是子view的isHidden并没有被设置为true。

```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    if (self.isHidden) {
        return nil;
    }
    return [self.subView hitTest:point withEvent:event];
}
```

上面的例子，父view里面重写了hitTest方法，如何加一个isHidden判断，否则还是会走到子view的hitTest方法，内部会进行跳转逻辑。

## converRect

今天在做一个需求的时候，需要知道页面中某个view在控制器view中的rect，开始想到的时取superView，然后取superView的frame。

后来发现其实可以使用covertRect方法来获取到某个view在外层view中的具体位置，这样就不用关心具体的层级了。

## transform

跑马灯的动画，可以通过设置transform动画来实现。固定起点和终点（负数）。

## 圆形进度

CAShapeLayer 搭配 UIBezierPath。

首先构建UIBezierPath，构建一个贝塞尔曲线方法如下：

{% highlight objc%}
UIBezierPath *path = [UIBezierPath bezierPathWithArcCenter:CGPointMake(centerX, centerY) radius:radius startAngle:(-0.5f*M_PI) endAngle:1.5f*M_PI clockwise:YES];
{% endhighlight %}


```
                      1.5π
         +-------------+--------------+
         |             |              |
         |             |              |
         |             |              |
         |             |              |
         |             |              |
     1π  +-------------+--------------+ 0(2)π
         |             |              |
         |             |              |
         |             |              |
         |             |              |
         |             |              |
         +-------------+--------------+
		              0.5π
```

将path作为shapeLayer的路径。进度的更新就是根据progress来更新shapeLayer的strokeEnd值，这个值是支持做做动画的，这样就有了进度的动画。

## 圆角 任意角任意大小

设置某两个方向的圆角，可以使用bezierPath创建一个path。如果是两个方向的圆角大小不一样，就得自己画一个path了。参考代码如下：

{% highlight objc%}
- (CGPathRef)createPathWithRect:(CGRect)rect
                        topLeft:(CGFloat)topLeft
                       topRight:(CGFloat)topRight
                     bottomLeft:(CGFloat)bottomLeft
                    bottomRight:(CGFloat)bottomRight {
    CGFloat minX = CGRectGetMinX(rect);
    CGFloat minY = CGRectGetMinY(rect);
    CGFloat maxX = CGRectGetMaxX(rect);
    CGFloat maxY = CGRectGetMaxY(rect);

    const CGFloat topLeftCenterX = minX + topLeft;
    const CGFloat topLeftCenterY = minY + topLeft;

    const CGFloat topRightCenterX = maxX - topRight;
    const CGFloat topRightCenterY = minY + topRight;

    const CGFloat bottomLeftCenterX = minX +  bottomLeft;
    const CGFloat bottomLeftCenterY = maxY - bottomLeft;

    const CGFloat bottomRightCenterX = maxX -  bottomRight;
    const CGFloat bottomRightCenterY = maxY - bottomRight;

    CGMutablePathRef path = CGPathCreateMutable();
    //顶 左
    CGPathAddArc(path, NULL, topLeftCenterX, topLeftCenterY,topLeft, M_PI, 3 * M_PI_2, NO);
    //顶 右
    CGPathAddArc(path, NULL, topRightCenterX , topRightCenterY, topRight, 3 * M_PI_2, 0, NO);
    //底 右
    CGPathAddArc(path, NULL, bottomRightCenterX, bottomRightCenterY, bottomRight,0, M_PI_2, NO);
    //底 左
    CGPathAddArc(path, NULL, bottomLeftCenterX, bottomLeftCenterY, bottomLeft, M_PI_2,M_PI, NO);
    CGPathCloseSubpath(path);
    return path;
}
{% endhighlight %}

# 导航栏隐藏

在开发一个页面的时候，预期是导航栏隐藏，但是容器的vc设置导航栏隐藏，然后设置kvo打断点看看，是哪里又把导航栏展示出来了。

结果是pageVC的childVC内部，展示了导航栏。

# 内存警告

做一个糖葫芦列表，其中有一个cell需要做动画，同时需要保证滑动不能打断动画，所以加了一个标记位，开始只在下拉刷新的时候重置标记位。config方法里面判断标记位，进行return操作。

触发内存警告的时候，cell新建了一个，此时需要config，但是标记位没有被重置，导致出现了空白。所以监听内存警告的通知，重置标记位。

发现UIApplicationDidReceiveMemoryWarningNotification 和 didReceiveMemoryWarning 不是同步的。

所以解决思路得换一下，之前得思路是找到触发缓存清理得方式，比如内存警告，然后在这个地方去重置，不过万一没有发送内存警告通知，其实还是会有问题。所以换个角度，在cellForRow处进行处理，判断cell是否重新创建，是得话重置，这样就不用关心引发的各个情况。

# image resize scale

image的scale，默认是1。如果本地图片（assets里面也好，bundle里面也好）文件名中带@3x，@2x这样，scale会变成3，2。另一种情况是自己使用cgimage的方式生成指定scale。

比如tabbar的图片需要网络中下载，此时就需要下载完成后指定scale，否则上传的是3倍分辨率的图，以像素的形式展示就会很大。

还有一种情况是，网络中下载的一些背景图，因为容器的尺寸会变，比如高度固定但是宽度相对屏幕宽度或者气泡的尺寸根据内容变化，此时需要进行拉伸，保证不变形。

{% highlight objc%}
UIEdgeInsets insets = {0, 40, 0, 40};
[image resizableImageWithCapInsets:insets resizingMode:UIImageResizingModeStretch];
{% endhighlight %}

这个时候下载下来的图片，需要指定scale，和本地的占位图的scale保持一致，否则默认scale是1，进行拉伸效果会不符合预期。

# UITextView

textview文字超过frame的时候可以滚动，今天遇到一个情况是，展示textview的时候会进行初始文字填充，此时textview不能滚动，需要输入一行内容才能触发滚动。

经过view debug查看，发现textview约束存在警告`scrollable content size is ambiguous for uitextview`。

解决警告是因为在textview内部添加了一个占位label，设置了左右顶部高度约束。当把占位label使用textview的父view添加，警告消失了，也可以正常滑动了。
