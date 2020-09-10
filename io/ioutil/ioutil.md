# IO工具

## 变量

```go
var Discard io.Writer = devNull(0)
```

## 函数

- `ReadAll(r io.Reader) ([]byte, error)`

从r中读取所有数据。重复调用也不会返回`EOF`错误，而是户返回`nil`。

- `ReadFile(filename string) ([]byte, error)`

从文件中读取所有数据。重复调用也不会返回EOF错误，而是户返回`nil`。

- `WriteFile(filename string, data []byte, perm os.FileMode) error`

将数据写入文件。如果文件已经存在，旧数据会清空。

- `ReadDir(dirname string) ([]os.FileInfo, error)`

返回文件夹下的所有项目。

- `NopCloser(r io.Reader) io.ReadCloser`

将`r`转换成`io.ReadCloser`

## 源码解析

- `sync.Pool`使用案例

```go
var blackHolePool = sync.Pool{
    // 定义临时对象创建方法
    New: func() interface{} {
        b := make([]byte, 8192)
        return &b
    },
}

type devNull int

func (devNull) ReadFrom(r io.Reader) (n int64, err error) {
    bufp := blackHolePool.Get().(*[]byte) // 获取一个临时对象
    readSize := 0
    for {
        readSize, err = r.Read(*bufp)
        n += int64(readSize)
        if err != nil {
            blackHolePool.Put(bufp) // 用完临时对象，重新放回
            if err == io.EOF {
                return n, nil
            }
            return
        }
    }
}
```