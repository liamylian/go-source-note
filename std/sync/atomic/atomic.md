# atomic

## 原子函数

### 替换为新值，并返回旧值。

- `SwapInt32(addr *int32, new int32) (old int32)`
- `SwapInt64(addr *int64, new int64) (old int64)`
- `SwapUint32(addr *uint32, new uint32) (old uint32)`
- `SwapUint64(addr *uint64, new uint64) (old uint64)`
- `SwapUintptr(addr *uintptr, new uintptr) (old uintptr)`
- `SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)`

### 比较旧值是否相等，并替换为新值

- `CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)`
- `CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)`
- `CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)`
- `CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)`
- `CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool`
- `CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)`

### 加减

- `AddInt32(addr *int32, delta int32) (new int32)`
- `AddUint32(addr *uint32, delta uint32) (new uint32)`
- `AddInt64(addr *int64, delta int64) (new int64)`
- `AddUint64(addr *uint64, delta uint64) (new uint64)`
- `AddUintptr(addr *uintptr, delta uintptr) (new uintptr)`

### 取数

- `LoadInt32(addr *int32) (val int32)`
- `LoadInt64(addr *int64) (val int64)`
- `LoadUint32(addr *uint32) (val uint32)`
- `LoadUint64(addr *uint64) (val uint64)`
- `LoadUintptr(addr *uintptr) (val uintptr`
- `LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)`

### 存数

- `StoreInt32(addr *int32, val int32)`
- `StoreInt64(addr *int64, val int64)`
- `StoreUint32(addr *uint32, val uint32)`
- `StoreUint64(addr *uint64, val uint64)`
- `StoreUintptr(addr *uintptr, val uintptr)`
- `StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)`
