# redux

一个单向数据流的结构。

## async

## thunk

## saga

`redux-saga` 也是一个中间件，他的作用在于将异步的操作放在了generator函数中，这样保证了action的纯洁。

## 客户端实践

正常情况一个控制器一个Store，此时redux主要负责的是异步相关的State的数据流动。如果是分页接口，state中增加page参数, 当分页接口有多个过滤条件，可以增加一个config state。请求下一页的action和第一次请求一样，action内部进行参数的组装。

另一种情况，多个页面共用一组state，举个例子，个人信息模块，一个展示，若干个修改。这个时候，创建一个静态的Store。

---

当控制器订阅Store的时候，当设计到分页的上拉和下拉的时候，update内部需要除了需要处理列表的刷新，还需要额外的处理loading状态，所以看上去就不那么清晰。
参看了React中的实现(connect)，component直接订阅store，也就是说和state相关的component都会去订阅state，所以每个component做的事情都分布在了各个component中，自然就清晰了很多。

# react-redux

使用redux我们需要自己调用store.subscribe、store.dispatch和store.getState这些方法，并且在state更新的时候，要自己重新触发render，业务逻辑复杂的话还是会有一点麻烦，所以redux的作者封装了一个专门给react使用的redux模块，就是react-redux。

首先是提供了一个Provider，进行包装最外层组件，将store放到Provider里面，这样整个组件都能获取到store。

[ref](https://juejin.cn/post/6847902216234369037)

# mobx

mobx 也是一个单向数据流的框架。

## state 

## action

## Derivations

大致看了下文档，mobx面向对象的方式相比redux要简单一些，用了类似Rx中的observable的机制来进行订阅刷新，避免了redux中手动去写connect。

# mobx-react

类似react-redux，提供了一个Provicer进行包装最外层组件。

observer进行包装组件，组件监听store变化，更新UI。inject则是用来注入组件想要关注当某几个store，而不必去取到整个store，类型connect函数。

# References

[https://github.com/ProtoTeam/blog/blob/master/201710/3.md](https://github.com/ProtoTeam/blog/blob/master/201710/3.md)
