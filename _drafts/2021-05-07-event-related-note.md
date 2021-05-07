
# 点击

当发生点击的时候，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。

app启动时候，runloop中注册了一个source来监听外界进程传递过来的事件。

SpringBoard 只接收按键等事件，随后通过mach port转发给需要的app 进程，此时runloop中注册的这个source1事件回调会被执行，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent，此时会调用UIApplication的`sendEvent`方法，将event传给UIWindow。

开始对window进行hitTest遍历，找到树形结构最底部的叶子节点view。

此时判断这个view是否包含gesture，将事件交给gesture进行识别，此时touchBegin/Move方法会同步处理，当gesture识别成功了，此时会调用touchCancel方法，执行手势的target/action。

此时判断这个view是否是UIControl的子类，如果是直接将事件交给UIControl完成。

UIControl会调用sendAction:to:forEvent:来将行为消息转发到UIApplication对象，再由UIApplication对象调用其sendAction:to:fromSender:forEvent:方法来将消息分发到指定的target上。

最后则是touchs相关方法被执行，如果叶子节点的view没有实现相关方法，这个时候沿着树形结构，从下往上寻找，直到window，UIApplication对象，如果没有被处理，则被丢弃。

# References

[ref](https://blog.gocy.tech/2016/11/19/iOS-touch-handling/)

[ref](https://blog.ibireme.com/2015/05/18/runloop/)

[ref](https://www.jianshu.com/p/c4d8082989e2)
