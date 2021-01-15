
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

# References

[https://www.mikeash.com/pyblog/friday-qa-2015-12-11-swift-weak-references.html](https://www.mikeash.com/pyblog/friday-qa-2015-12-11-swift-weak-references.html)
[Object lifecycle state machine](https://github.com/apple/swift/blob/main/stdlib/public/SwiftShims/RefCount.h)
[https://www.mikeash.com/pyblog/friday-qa-2017-09-22-swift-4-weak-references.html](https://www.mikeash.com/pyblog/friday-qa-2017-09-22-swift-4-weak-references.html)
