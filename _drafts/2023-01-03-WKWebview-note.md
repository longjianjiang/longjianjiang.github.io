

# cookie

cookie的注入iOS11以后需要将 NSHTTPCookieStorage中的Cookie信息复制到 WKHTTPCookieStore 中，以此来达到 WKWebView中注入Cookie 的目的。

NSURLRequest也提供了添加cookie的方法，访问某个页面可以带上需要注入的cookie。

cookie的删除iOS11以后可以使用WKHTTPCookieStore的deleteCookie: completionHandler: 方法来操作。

[ref](https://www.jianshu.com/p/4b51e44f9e90)
