
# WebviewJSBridge

## Native -> JS

当JS文件里`registerHandler`后，Native可以`callHandler`来调用到JS。过程如下：

1. 根据handlerName，data（传递给JS的参数），callback（JS回调Native），callback使用一个callbackId来进行标记，同时将callback存到dict中，最后将前面三个信息组装成一个字典；

2. 用前面的字典作为参数，拼接一段JS，调用`evaluateJavaScript`执行；

3. JS收到字典进行解析，可以拿到前面三个参数，根据handlerName去Dict中找到注册时的那个handler，当callbackId存在的时候，JS还需要构造一个回调function来进行一次回调，JS回调Native就是通过发请求，通过修改iframe的src来触发wkwebview的`decidePolicyForNavigationAction`方法，这个方法则对scheme和host进行了过滤。同时JS将一条回调的message存到队列中，Native拦截的实现就是去执行JS的一个方法，取到存的那条message，根据callbackId取到callHandler传的block，将JS传递过来的参数进行调用。

## JS -> Native

同样的Native中`registerHandler`后，JS可以`callHandler`来调用Native。过程如下：

1. 还是最多有三个参数，handlerName，callback，data。当存在callback的时候，依然时使用Id来标记callback，构建一个message，存到队列，改变src触发Native回调；

2. Native拦截后，执行JS，JS则时简单将刚存的message进行返回，此时Native根据message里的callbackId来构建一个回调JS的callback，最后Native根据message进行中的handler进行找到注册的handler进行处理;

3. Native创建的handler中，到执行到callback时，则Native构造一个message作为参数，构造一段JS，调用`evaluateJavaScript`执行，最后JS会拿到Native传递的参数取调用自己的callback；

---

核心还是webview的`evaluateJavaScript`来进行中转。JS端和Native端通过responseId来判断是对方回调自己，此时直接根据Id取到callback来进行调用即可。callbackId的时候则是去调用对方。

# References

[https://github.com/marcuswestin/WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)
