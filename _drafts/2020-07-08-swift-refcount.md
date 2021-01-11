
# HeapObject

```cpp
struct HeapObject {
	RefCount refCount;
	HeapMetaData const *metadata;
};
```

# RefCount

[LIVE]

deinit -> [DEINITING]

canBeFreedNow() => false -> [DEINITED] -> [FREED] -> [DEAD]

canBeFreedNow() => true -> [DEAD]

[ref](https://zhongwuzw.github.io/2017/06/17/Swift%E4%B9%8BWeak%E5%BC%95%E7%94%A8/)
