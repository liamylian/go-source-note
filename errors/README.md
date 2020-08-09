# 错误处理

## 依赖类型

- `builtin.error`

内置基本接口。

```go
type error interface {
	Error() string
}
```

## 函数

- `New()`

创建基本错误（errorString类型）

```go
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

- `Unwrap(err error) error`

获取上层错误，调用`err`的`Unwrap`的方法获取上层错误，若`err`未实现`Unwrap`则返回nil。

- `fmt.Errorf(format string, a ...interface{})`

用于生成带底层`error`的`error`。

例：
```go
    fmt.Errorf("%w", err)
```

- `Is(err, target error) bool`

判断`err`与`target`是否匹配。

匹配条件：

1. 完全相等
2. err如果实现了`Is(interface{}) bool`，而且err.Is(target)返回`true`，则返回`ture`
3. err = err.Unwrap，若err=nil，返回`false`，否则从`1`开始重新匹配。

- `As(err error, target interface{}) bool`

将`err`赋值给`target`。

流程：

1. `target`需为非空interface指正，或实现了`error`，否则panic。
2. err如果实现了`As(interface{}) bool`，而且err.As(target)返回`true`，则返回`ture`
3. err = err.Unwrap，若err=nil，返回`false`，否则从`1`开始重新匹配。
