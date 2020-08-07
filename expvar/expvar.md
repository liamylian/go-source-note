# expvar

局公共变量库，设置或修改这些公共变量的操作是原子的。这些公共变量可以通过HTTP接口`/debug/vars`查看。

## 接口

```go
type Var interface {
	// 必须返回JSON类型数值
	String() string
}
```

## 类型

- Int

|方法|描述|
|---|---|
|Add(delta int64)|原子增加|
|Set(value int64)|原子赋值|
|Value() int64|获取值|
|String() string|返回JSON数值|

- Float

|方法|描述|
|---|---|
|Add(delta float64)|原子增加|
|Set(value float64)|原子赋值|
|Value() float64|获取值|
|String() string|返回JSON数值|

- String

|方法|描述|
|---|---|
|Set(value string)|原子赋值|
|Value() string|获取值|
|String() string|返回JSON数值|

- Func

|方法|描述|
|---|---|
|Value() interface{}|获取值|
|String() string|返回JSON数值|

- Map

|方法|描述|
|---|---|
|Init()|清空记录|
|Get(key string) Var|获取变量|
|Set(key string, av Var)|添加变量|
|String() string|返回JSON数值|
|Add(key string, delta int64)|原子增加|
|AddFloat(key string, delta float64)|原子增加|
|Delete(key string)|移除变量|
|Do(f func(KeyValue))|遍历变量|


## 函数

- `Publish(name string, v Var)`

发布变量。

- `Get(name string) Var`

获取变量。

- `NewInt(name string) *Int`

新建`Int`变量。

- `NewFloat(name string) *Float`

新建`Float`变量。

- `NewMap(name string) *Map`

新建`Map`变量。

- `NewString(name string) *String`

新建`String`变量。

- `Do(f func(KeyValue))`

遍历已发布变量。

- `Handler()`

返回`HTTP`处理函数。
