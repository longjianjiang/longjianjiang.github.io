

# 实现原理

- RAC

调用`rac_signalForSelector`后，会生成一个receiver对象的子类，swizzle子类的`forwardInvocation:`，`respondsToSelector:`, `methodSignatureForSelector:`, `class` 方法。其中最重要的是`forwardInvocation:`。子类创建完成后，将receiver改成子类类型，同时往receiver的关联对象中存储了子类信息。

创建一个RACSubject对象，将其存储到receiver的关联对象中。当receiver有selector的实现时，这个时候会为子类中添加一个`rac_alias_selector`的实现来保存原先selector，然后将原selector的实现替换为`_objc_msgForward`实现，也就是转发。

所以当receiver调用selector时，因为此时receiver是RAC新建子类类型，所以会走到替换后的消息转发方法中，该方法首先取到之前新添加的`rac_alias_selector`进行执行，也就是selector的实现，之后将该方法的参数进行sendNext。

另外，`RACDelegateProxy`作为一个delegate代理，以UITextView的rac_textSignal为例。分类中有一个`RACDelegateProxy`的实例`rac_delegateProxy`，用它调用`rac_signalForSelector`，效果如上所示，`textViewDidChange:`被调用时，转发中可以拦截到，同样的如果textView有delegate，则会先执行delegate实现的`textViewDidChange:`方法，之后转发中拿到text进行sendNext。
