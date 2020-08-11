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

判断`err`与`target`是否同类（或包含`target`）。

相类条件：

1. 完全相等
2. err如果实现了`Is(interface{}) bool`，而且err.Is(target)返回`true`，则返回`ture`
3. err = err.Unwrap，若err=nil，返回`false`，否则从`1`开始重新匹配。

- `As(err error, target interface{}) bool`

将`err`赋值给`target`。

流程：

1. `target`需为非空interface指正，或实现了`error`，否则panic。
2. err如果实现了`As(interface{}) bool`，而且err.As(target)返回`true`，则返回`ture`
3. err = err.Unwrap，若err=nil，返回`false`，否则从`1`开始重新匹配。

## 源码解读

```go
// 获取底层错误
func Unwrap(err error) error {
    // 如果`err`实现了`UnWrap() error`方法，那么返回该函数返回值；否则返回nil。
	u, ok := err.(interface {
		Unwrap() error
	})
	if !ok {
		return nil
	}
	return u.Unwrap()
}

// 判断`err`与`target`是否同类（或包含`target`）
func Is(err, target error) bool {
	if target == nil {
		return err == target
	}

	isComparable := reflectlite.TypeOf(target).Comparable()
	for {
		if isComparable && err == target {
            // 相等即同类
			return true
		}
		if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
            // `err`实现了`Is(error) bool`方法，而且`err.Is(target)`返回`true，返回
			return true
		}
        // 获取下层错误，并重新判断
		if err = Unwrap(err); err == nil {
			return false
		}
	}
}

// 将`err`（或包含的`error`）赋值给`target`。
func As(err error, target interface{}) bool {
	if target == nil {
        // `target`不能为nil
		panic("errors: target cannot be nil")
	}
	val := reflectlite.ValueOf(target)
	typ := val.Type()
	if typ.Kind() != reflectlite.Ptr || val.IsNil() {
        // `target`必须为非空指针
		panic("errors: target must be a non-nil pointer")
	}
	if e := typ.Elem(); e.Kind() != reflectlite.Interface && !e.Implements(errorType) {
        // `*target`必须为接口类型，或者实现了`error`接口（函数，自定义类型等也可以实现接口）
		panic("errors: *target must be interface or implement error")
	}
	targetType := typ.Elem()
	for err != nil {
		if reflectlite.TypeOf(err).AssignableTo(targetType) {
            // 如果`err`可赋值给`target`变量的类型，赋值并返回`true`
			val.Elem().Set(reflectlite.ValueOf(err))
			return true
		}
		if x, ok := err.(interface{ As(interface{}) bool }); ok && x.As(target) {
            // 如果`err`实现了`As(interface{}) bool`接口，并且使用该方法赋值成功，则返回`true`
			return true
		}
        // 获取下层`error`，并继续重试
		err = Unwrap(err)
	}
	return false
}
```