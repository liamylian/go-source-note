# 全局公共变量

全局公共变量库，设置或修改这些公共变量的操作是原子的。这些公共变量可以通过HTTP接口`/debug/vars`查看。

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
|Add(delta int64)|原子加减|
|Set(value int64)|原子赋值|
|Value() int64|获取值|
|String() string|返回JSON数值|

- Float

|方法|描述|
|---|---|
|Add(delta float64)|原子加减|
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

## 源码解读

### `float64`类型原子加减

`atomic`库仅支持整数类型原子加减，原子加载，原子赋值和原子交换。要对其他类型进行原子操作，需要先做📨类型转换。

```go
type Float struct {
    f uint64
}

func (v *Float) Add(delta float64) {
    for {
        // 加载当前整数值
        cur := atomic.LoadUint64(&v.f)
        // 将当前数值转化为浮点类型
        curVal := math.Float64frombits(cur)
        // 对浮点数进行加减
        nxtVal := curVal + delta
        // 将计算结果从新转化为整数值
        nxt := math.Float64bits(nxtVal)
        // 使用`CPU`原子指令`比较并交换`进行赋值，如果失败重试，直到成功
        if atomic.CompareAndSwapUint64(&v.f, cur, nxt) {
            return
        }
    }
}
```

### 高并发有序Map实现（读多写少）

使用sync.Map和有序健名数组实现。

```go
type Map struct {
    m      sync.Map // map[string]Var
    keysMu sync.RWMutex // 键名数组锁
    keys   []string // 排序的健名
}

// 添加健名，并重新排序健名数组
func (v *Map) addKey(key string) {
    // 加琐
    v.keysMu.Lock()
    defer v.keysMu.Unlock()

    // 查找并插入健名
    if i := sort.SearchStrings(v.keys, key); i >= len(v.keys) {
        v.keys = append(v.keys, key)
    } else if v.keys[i] != key {
        v.keys = append(v.keys, "")
        copy(v.keys[i+1:], v.keys[i:])
        v.keys[i] = key
    }
}

func (v *Map) Set(key string, av Var) {
    // 在存储元素前，先检查元素是否已经存在，即先`Load`然后再`LoadOrStore`。
    // LoadOrStore causes the key interface to escape even on the Load path.（TODO: 翻译）

    // 先检查元素否已经存在
    if _, ok := v.m.Load(key); !ok {
        // 尝试存储元素，如果失败则说明元素已存在
        if _, dup := v.m.LoadOrStore(key, av); !dup {
            // 元素不存在才添加健名
            v.addKey(key)
            return
        }
    }

    // 保存元素
    v.m.Store(key, av)
}
```