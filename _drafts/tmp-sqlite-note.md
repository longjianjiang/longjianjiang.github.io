
# WAL

sqlite Write-ahead logging. 一种用来实现事务原子性的机制。

在WAL之前，sqlite使用rollback journal机制实现原子事务。当修改数据库之前，先将修改所在分页的数据备份在某个地方，然后将修改写入数据库，当事务失败，则将备份拷贝回来，撤销修改；对应的事务成功，删除备份数据，提交修改。

WAL则不直接将修改写入数据库，而是写入另外一个WAL文件中，如果事务失败，WAL中的记录会被忽略，撤销修改；对应的事务成功，WAL中的记录会在某个时间点被写入到数据库中。同步WAL和数据库的行为被成为checkpoint。默认是当WAL累计到1000页修改的时候，sqlite自动进行同步，也可以手动调用接口进行同步。

每次进行读操作都会去WAL中进行搜索，找到当前最后的写入点，称为end mark记录下来，忽略之后写入的数据。

然后判断要读的数据是否在WAL中，如果存在直接从WAL中读取，否则去数据库中进行读取。

进行写操作的时候，只是简单的往WAL后添加新的内容，因为有end mark的存在，所以读写可以同时进行。不过因为WAL文件只有一个所以每次只能有一个写操作。

```
		mark
  |  |	 |
  V  V   |
---------+-----------------
         |
---------+-----------------
		 V
```

如上图在进行同步的时候，当遇到end mark的时候，则需要停止同步，因为此时end mark后的数据正在被某个reader使用，如果进行同步等于重写了数据库中的数据，所以此时会进行一次记录本次同步的距离，下次继续同步。

# 事务模型


# References

[https://www.sqlite.org/wal.html](https://www.sqlite.org/wal.html)

[https://zhangbuhuai.com/post/sqlite.html](https://zhangbuhuai.com/post/sqlite.html)
