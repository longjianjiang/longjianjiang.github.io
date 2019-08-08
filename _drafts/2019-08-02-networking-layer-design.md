

> 本文，笔者记录下网络层设计的一些点；

# URLSession

# AFNetworking

# MyNetworking

一般App的网络层不会直接使用URLSession或者AFNetworking提供的API，需要进行一层封装。

这层封装有如下两个必要性：

1> 减少重复代码的编写；

2> 额外增加一层，有助于添加其他功能；

下面笔者列出一些“添加的功能”：

- 拦截器

可能在请求发送之前，请求之后需要一些事件的处理，比如loading。这个时候通过拦截器这种AOP的方法，让这些请求无关的代码可以隔离开来。

- reformer

请求返回的数据通常不是立即可以使用的，当越到一个API给不同View去展示，一个View展示来自不同API的数据这类情况的时候，不可避免的需要进行转化。

所谓reformer这层就是用来做这件事的，这样可以将转化的逻辑独立出来，即减轻控制器的负担，也可以进行复用。

- mock

有的时候API返回的是正确的数据，但是我们想测试异常情况，这个时候可以手动创建一份假的返回来进行模拟。

---

除了基本的功能，剩下的我想应该就是安全机制，优化机制了。

# References

[https://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html](https://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html)

[https://github.com/Moya/Moya](https://github.com/Moya/Moya)
