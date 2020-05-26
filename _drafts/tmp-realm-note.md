# Object

realm中的object就相当于是表；

# CRUD

更新操作需要开启一个事务去进行；

事务不允许嵌套，开启一个事务后开启另一个事务会抛异常。

如果需要连续执行两次write操作，需要使用`beginWrite`和`commitWrite`进行包装；

# Thread

跨线程需要使用相同的配置创建一个新的realm去使用；

# Test

单元测试使用内存realm；

# References

[ref](https://realm.io/docs/swift/latest/)
