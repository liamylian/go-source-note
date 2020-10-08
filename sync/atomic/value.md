# Value

## 类型

```go
type Value struct {
	v interface{}
}
```

## Value方法

- `Load() (x interface{})`

返回最近一次`Store`的数据。

- `Store(x interface{})`

存放数据。

## 接口内部实现

```go
// ifaceWords is interface{} internal representation.
type ifaceWords struct {
	typ  unsafe.Pointer // 类型指针
	data unsafe.Pointer // 数据指针
}
```