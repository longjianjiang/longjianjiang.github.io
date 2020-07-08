
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
