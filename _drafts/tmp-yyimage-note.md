

# 图片解码

一般我们说的图片例如png，gif都是一种压缩格式，要想实际显示到屏幕上需要一个解码的过程，解码完成后可以拿到图片的bitmap信息，CGImage或者CIImage。将bitmap交给CA进行渲染，从而进行展示。

当我们用`imageWithName`方法进行加载图片的时候，解码的过程是在主线程进行的，因此一定程度上会导致卡顿。

Apple提供了ImageIO进行图片的编解码。下面笔者以gif为例来简单给出一个解码的操作：

{% highlight swift %}
func decodeImageWithData(_ data: Data) -> UIImage? {
	let source = CGImageSourceCreateWithData(data as CFData, nil)
	guard let imageSource = source else {
		return nil
	}

	let frameCount = CGImageSourceGetCount(imageSource)
	if frameCount == 0 { return nil }

	var images: [UIImage] = []
	var totalDuration = 0.0

	for i in 0..<frameCount {
		guard let frameProperties = CGImageSourceCopyPropertiesAtIndex(imageSource, i, nil) as? [String: Any] else {
			return nil
		}
		guard let gifInfo = frameProperties[kCGImagePropertyGIFDictionary as String] as? [String: Any] else {
			return nil
		}
		let unclampedDelayTime = gifInfo[kCGImagePropertyGIFUnclampedDelayTime as String] as? NSNumber
		var duration = 0.0
		if let delayTime = unclampedDelayTime {
			duration = delayTime.doubleValue
		}
		let imageRef = CGImageSourceCreateImageAtIndex(imageSource, i, nil)!
		let imageOrientation: UIImage.Orientation = .up
		let image = UIImage(cgImage: imageRef, scale: UIScreen.main.scale, orientation: imageOrientation)
		totalDuration += duration
		images.append(image)
	}
	return UIImage.animatedImage(with: images, duration: totalDuration)
}
{% endhighlight %}

可以看到首先根据data创建一个source，根据source获取gif的帧数，根据每一帧使用CGImageSourceCreateImageAtIndex得到CGImage，拿到CGImage和总时长创建UIImage。

## forceDecode

ImageIO采用了惰性解码的方式，`CGImageSourceCreateImageAtIndex`方法返回的CGImage其实只是一个包含元信息的空Image。

也就是说图片的解码操作在UIKit的渲染pipeline中依然是发生在主线程的，当加载大图的时候，第一次会有帧率下降，第二次则正常因为存在复用。

所以为了forceDecode就是提前让ImageIO进行解码，这样我拿到的CGImage是解码完成的，当UIKit进行渲染的时候就不需要额外的解码过程了。

具体做法是新建一个画布CGContext，调用`CGContextDrawImage`进行绘制，这个方法会使ImageIO立即解码。具体实现可以参考`YYCGImageCreateDecodedCopy`。

苹果没有使用ForceDecode原因在于，早期iOS设备的内存有限，如果大图进行ForceDecode会增加内存的压力，为了内存开销稳定，采取了这套惰性解码。

## webp

webp没有被iOS系统解码器支持，所以只能使用CPU进行软件解码，一般会提供一个获取bitmap buffter的接口，对于webp解码没有使用惰性解码而是直接返回一个含有bitmap buffer的CGImage。

webp同样也使用到了forceDecode。`CGContextDrawImage`调用后，会将bitmap buffer提交到render server，这样可以省去渲染时的内存拷贝操作。

# References

[ref1](https://dreampiggy.com/2019/01/18/%E4%B8%BB%E6%B5%81%E5%9B%BE%E7%89%87%E5%8A%A0%E8%BD%BD%E5%BA%93%E6%89%80%E4%BD%BF%E7%94%A8%E7%9A%84%E9%A2%84%E8%A7%A3%E7%A0%81%E7%A9%B6%E7%AB%9F%E5%B9%B2%E4%BA%86%E4%BB%80%E4%B9%88/)
