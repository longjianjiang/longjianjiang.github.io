# 前端基础笔记

本文记录笔者整理的一些前端的知识。

## dom

所谓dom(document object model) 其实是用来操作html的api。整个html可以被看作是一个dom树，<div></div> 这些标签都可以看作是树的一个节点。

document 是dom树的根节点，包含了标题(document.title)，url(document.URL)等属性，子节点(document.getElementById("xx"))，可以直接在JS中访问。

## bom

所谓bom(browser object model) 是用来控制浏览器的api。

location 是bom中的一个对象，比如常用的`location.href = "xx"`。

window 也是bom中的一个对象。

## event

和Cocoa里面的target/action机制类似，event机制是浏览器提供的一种事件分发的机制，可以创建一个event，同样的页存在一些定义好的事件（鼠标悬停）。

可以监听和解除监听对应的event。

```js
EventTarget.dispatchEvent()

EventTarget.addEventListener(type, listener [, useCapture])

EventTarget.removeEventListener(type, listener [, useCapture])
```

docment，window，element都是eventTarget。

event的传递有三个阶段，捕获阶段，目标阶段，冒泡阶段。

```
window

document

div
```

当div触发了event后，捕获阶段会先从window->document，然后到div本身。然后document->window，以相反方向完成冒泡阶段。

`useCapture`的默认是false，也就是默认没有使用捕获阶段，所以当div触发事件，会首先通知到div。

下面是一个实际的例子：

```js
var doc = document
var readyEvent = doc.createEvent('Events')
readyEvent.initEvent('DXYJSBridgeReady')
readyEvent.bridge = DXYJSBridge
doc.dispatchEvent(readyEvent)

document.addEventListener('DXYJSBridgeReady', () => {
console.log('DXYJSBridge初始化完成');
});
```

更多[参考](https://segmentfault.com/a/1190000018266823)

## jsbridge

所谓jsbridge就是注入到一段脚本到webview上，一般是一个对象，这个对象有一个invoke方法用来给js进行调用native的方法。然后webview在收到脚本回调中根据name进行解析去分发处理，处理好了，调用`evaluateJavaScript`将结果回调给js。

```
window.webkit.messageHandlers.JSBridge.postMessage(msg)

open func createWebViewConfiguration() -> WKWebViewConfiguration {

	let configuration = WKWebViewConfiguration()
	configuration.allowsInlineMediaPlayback = true

	let script = WKUserScript(source: DXYWKWebViewController.JSCode, injectionTime: .atDocumentEnd, forMainFrameOnly: false)

	let userContentController = WKUserContentController()
	userContentController.addUserScript(script)

	let scriptHandler = WeakScriptMessageHandler(realHandler: self)
	userContentController.add(scriptHandler, name: "JSBridge")

	configuration.userContentController = userContentController
	return configuration
}
``

JSBridge 就是向webview中注入的内容。`webkit.messageHandlers.xx.PostMessage` 是webkit提供的js向native发送消息的api。

postMessage 后，`WKScriptMessageHandler` 的方法会触发：

```swift
open func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
	guard let message = message.body as? [String: AnyObject] else {
		return
	}
	self.dispatchMessageFromJavaScript(self.webView, message: message)
}
```

这个方法在进行handler的分发和调用`evaluateJavaScript` 将结果回调给js。

[ref](https://juejin.im/post/6844904164586160142#heading-4)

## hybrid

TODO

[ref](https://www.jianshu.com/p/efb4f93b10de)
