---
layout: post
title:  "【Reative】学习笔记第三弹(Observable & Subject)"
date:   2019-05-13
excerpt:  "本文是笔者学习Reative的第三篇文章，主要介绍Observable和Subject"
tag:
- Reative
comments: true
---

本文笔者将介绍在之前介绍的冷信号和热信号，以RxSwift中的实现为例。

# Observable

Observable是RxSwift中所谓的冷信号，通常用来来执行某一个具体的任务，将执行的过程，从开始到结束这一整个过程进行分发给订阅者。

下面笔者给出一个例子：

{% highlight swift %}
let urlStr = "https://com.longjianjiang.com/image/test.png"
guard let url = URL.init(string: urlStr) else { return }
let observable = Observable<Data>.create { observer in
	let sessionTask = URLSession.shared.dataTask(with: url, completionHandler: { (data, response, error) in
		if let data = data {
			observer.onNext(data)
			observer.onCompleted()
		}
		if let error = error {
			observer.onError(error)
		}
	})
	sessionTask.resume()

	return Disposables.create {
		sessionTask.cancel()
	}
}
observable
	.map { UIImage.init(data: $0) }
	.observeOn(MainScheduler.instance)
	.subscribe(onNext: { image in
		self.imgViewOne.image = image
	})
	.disposed(by: bag)
{% endhighlight %}

将一个网络请求进行表示成一个冷信号，订阅者可以拿到请求的data后，继续做下一步的操作，同时可以切换执行的线程，一系列的链式调用完成了下载图片到展示，可以发现代码看上去还是比较简洁的。

下面笔者将分析Observable在RxSwift中的实现。

## Observable Protocol 

{% highlight swift %}
public protocol ObservableConvertibleType {
    associatedtype Element

    func asObservable() -> Observable<Element>
}
public protocol ObservableType: ObservableConvertibleType {
    func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element
}
{% endhighlight %}


# Subject

# 总结

# References

{% highlight swift %}
{% endhighlight %}
