# mulit

## 类型

```go
type multiReader struct {
    readers []Reader
}

type multiWriter struct {
    writers []Writer
}
```

## 函数

- `MultiReader(readers ...Reader) Reader`

合并Reader

- `MultiWriter(writers ...Writer) Writer`

多路Writer

## `multiReader`方法

`Read(p []byte) (n int, err error)`

从多个Reader依次读取数据，直到每一个Reader都返回Eof。

## `multiWriter`方法

`Write(p []byte) (n int, err error)`

依次将数据写入多个Writer。

`WriteString(s string) (n int, err error)`

依次将字符串数据写入多个Writer。
