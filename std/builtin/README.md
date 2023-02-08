# 内置类型及方法

## 类型

- uint
- uint8
- uint16
- uint32
- uint64
- int
- int8
- int16
- int32
- int64
- float32
- float64
- complex64
- complex128
- string
- uintptr
- byte
- rune
- nil

## 内部标识

|类型|返回值|
|---|---|
|`Type`, `Type1`|类型标识|
|`IntegerType`|整数标识|
|`FloatType`|浮点数标识|
|`ComplexType`|复数标识|

## 函数

- `append(slice []Type, elems ...Type) []Type`

往切片末尾添加元素，并返回更新后的切片，返回的底层数组可能为新数组（容量不足）。
一般需要保存返回值。

- `copy(dst, src []Type) int`

拷贝切片数据，并返回拷贝元素的个数。
仅拷贝min(len(dst), len(src))个数的元素。

- `delete(m map[Type]Type1, key Type)`

从字典里删除元素。

- `len(v Type) int`

返回类型长度。

|类型|返回值|
|---|---|
|数组，数组指针|元素个数|
|切片|元素个数|
|管道|缓冲元素个数|

- `cap(v Type) int`

返回类型容量。

|类型|返回值|
|---|---|
|数组，数组指针|元素个数|
|切片|最大元素个数|
|管道|最大缓冲元素个数|

- `make(t Type, size ...IntegerType) Type`

分配并初始化切片，字典，管道对象。返回的类型不同于`new`返回指针，而是返回具体类型。

|类型|size[0]|size[1]|描述|
|---|---|---|---|
|切片|切片元素个数（可选）|切片元素容量（可选）|size[1]必须大于size[0]|
|字典|最少元素数量（可选）|-|分配至少能容下size[0]个元素的空间|
|管道|缓冲元素个数（可选）|-|size[0]=0，则不缓冲|

- `new(Type) *Type`

分配类型空间，并返回指针。

- `complex(r, i FloatType) ComplexType`

构造复数类型。

- `real(c ComplexType) FloatType`

获取复数实部。

- `imag(c ComplexType) FloatType`

获取复数虚部。

- `func close(c chan<- Type)`

关闭管道（必须为双向或，只发送）。当最后一个元素被接收后，后续不会再阻塞，并返回`空值`及`false`。

- `panic(v interface{})`

运行当前函数F的`defer`函数，终止并返回调用函数G。调用函数G同F，运行`defer`函数，终止并返回调用函数X。
函数X以此类推，直到协程停止。可通过`recover`停止流程。

- `recover() interface{}`

捕获并停止`panic`流程，如果返回`nil`，标识没有`panic`。

- `print(args ...Type)`

输出到标准输出，函数可能会被淘汰。

- `println(args ...Type)`

输出到标准输出，函数可能会被淘汰。

## 接口

- 错误类型`error`

```go
type error interface {
    Error() string
}
```
