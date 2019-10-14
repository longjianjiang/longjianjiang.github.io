
# redux

# async

# thunk

# saga

`redux-saga` 也是一个中间件，他的作用在于将异步的操作放在了generator函数中，这样保证了action的纯洁。

# 实践

正常情况一个控制器一个Store，此时redux主要负责的是异步相关的State的数据流动。如果是分页接口，state中增加page参数，请求下一页的action和第一次请求一样，action内部进行参数的组装。

另一种情况，多个页面共用一组state，举个例子，个人信息模块，一个展示，若干个修改。这个时候，创建一个静态的Store。

# References

[https://github.com/ProtoTeam/blog/blob/master/201710/3.md](https://github.com/ProtoTeam/blog/blob/master/201710/3.md)
