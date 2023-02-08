# flag

## 类型

- 接口

```go
// 选项值
type Value interface {
    String() string
    Set(string) error
}

// 用于表示选项可以没有值，即可以不用指定`=value`
type boolFlag interface {
    Value
    IsBoolFlag() bool
}

type Getter interface {
    Value
    Get() interface{}
}
```

- 结构体

```go
// 选项组
type FlagSet struct {
    Usage func()

    name          string
    parsed        bool
    actual        map[string]*Flag
    formal        map[string]*Flag
    args          []string // 选项之后的参数
    errorHandling ErrorHandling
    output        io.Writer // `nil`即为`stderr`
}

// 选项
type Flag struct {
    Name     string 
    Usage    string
    Value    Value 
    DefValue string 
}
```

- 实现类型

|类型|名称|描述|
|---|---|---|
|boolValue|||
|intValue|||
|int64Value|||
|uintValue|||
|uint64Value|||
|stringValue|||
|float64Value|||
|durationValue|||

## 函数

- `VisitAll(fn func(*Flag))`

按字母顺序遍历所有选项。

- `Visit(fn func(*Flag))`

按字母顺序遍历已赋值选项。

- `func Lookup(name string) *Flag `

按名称返回选项。

- `Set(name, value string) error`

设置选项值。

- `UnquoteUsage(flag *Flag) (name string, usage string)`

返回去除引号的用法。

- `PrintDefaults()`

输出所有定义选项的默认值。

- `NFlag() int`

返回设置的选项个数。

- `func Arg(i int) string`

返回第`i`个参数。

- `NArg() int`

返回参数个数。

- `NewFlagSet(name string, errorHandling ErrorHandling) *FlagSet`

创建选项组。

- `Parse()`

从`os.Args[1:]`解析选项。

- `BoolVar(p *bool, name string, value bool, usage string)`

定义布尔选项。

- `Bool(name string, value bool, usage string) *bool`

定义布尔选项，并返回布尔变量指针。

- `IntVar(p *int, name string, value int, usage string)`

定义Int选项。

- `Int(name string, value int, usage string) *int`

定义Int选项，并返回变量指针。

- `Int64Var(p *int64, name string, value int64, usage string)`

定义Int64选项。

- `Int64(name string, value int64, usage string) *int64`

定义Int64选项，并返回Int64变量指针。

- `UintVar(p *uint, name string, value uint, usage string)`

定义Uint选项。

- `Uint(name string, value uint, usage string) *uint`

定义Uint选项，并返回Uint变量指针。

- `Uint64Var(p *uint64, name string, value uint64, usage string)`

定义Uint64选项。

- `Uint64(name string, value uint64, usage string) *uint64`

定义Uint64选项，并返回Uint64变量指针。

- `StringVar(p *string, name string, value string, usage string)`

定义字符串选项。

- `String(name string, value string, usage string) *string`

定义字符串选项，并返回字符串变量指针。

- `Float64Var(p *float64, name string, value float64, usage string)`

定义Float64选项。

- `Float64(name string, value float64, usage string) *float64`

定义Float64选项，并返回Float64变量指针。

- `DurationVar(p *time.Duration, name string, value time.Duration, usage string)`

定义时间段选项。

- `Duration(name string, value time.Duration, usage string) *time.Duration`

定义时间段选项，并返回时间段变量指针。

- `Var(value Value, name string, usage string)`

定义自定义选项。

## 源码解析

```go
// 判断字符串值是否为选项的空值
func isZeroValue(flag *Flag, value string) bool {
    // 创建选项空值，转化为字符串，并与`value`进行比较
    typ := reflect.TypeOf(flag.Value)
    var z reflect.Value
    if typ.Kind() == reflect.Ptr {
        z = reflect.New(typ.Elem())
    } else {
        z = reflect.Zero(typ)
    }
    return value == z.Interface().(Value).String()
}
```
