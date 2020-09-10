# 管道

Pipe适用于多个读取按写入顺序消费单个写入数据的场景。每次写入数据后都将被阻塞直到数据被完整读取。
不管是并行写入还是并行读取都是安全的，数据拷贝过程没有使用缓存。

## 类型

```go
// PipeReader, PipeWriter底层实现
type pipe struct {
    wrMu sync.Mutex // Serializes Write operations
    wrCh chan []byte
    rdCh chan int

    once sync.Once // Protects closing done
    done chan struct{}
    rerr onceError
    werr onceError
}

// 读管道
type PipeReader struct {
    p *pipe
}

// 写管道
type PipeWriter struct {
    p *pipe
}
```

## 函数

- `Pipe() (*PipeReader, *PipeWriter)`

创建管道。

## `pipe`方法

- `Read(b []byte) (n int, err error)`

读取数据。

- `CloseRead(err error) error`

关闭读。

- `Write(b []byte) (n int, err error)`

写数据。

- `CloseWrite(err error) error`

关闭写。

## `PipeReader`方法

- `Read(data []byte) (n int, err error) `

读数据。

- `Close() error`

关闭。

- `CloseWithError(err error) error`

出错关闭。

## `PipeWriter`方法

- `Write(data []byte) (n int, err error)`

写数据。

- `Close() error`

关闭。

- `CloseWithError(err error) erro`

出错关闭。