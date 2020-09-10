# IO

## 常量

|常量|数值|描述|
|---|---|---|
|SeekStart|0|从文件头开始|
|SeekCurrent|1|从当前位置开始|
|SeekEnd|2|从文件尾开始|

## 变量

|变量|描述|
|---|---|
|ErrShortWrite|写入数据不足|
|ErrShortBuffer|读取到的数据不足|
|EOF|文件尾|
|ErrUnexpectedEOF|读取固定长度数据中时，遇到文件尾|
|ErrNoProgress|尝试多次，任然读取不到数据|

## 接口

```go
type Reader interface {
    // 读取最多len(p)字节到p。即使读取到的数据小于n，也不代表已经读取完。
    // 
    // 文件内容读取完时，可能不会立刻返回(n, EOF)，而是可能在下次才返回(0, EOF)
    //
    // 当n>0时，应当先处理数据，再对err进行判断处理。
    Read(p []byte) (n int, err error)
}

type Writer interface {
    // 写len(p)字节数据。如果写入数据不足len(p)将会返回错误。
    Write(p []byte) (n int, err error)
}

type Closer interface {
    // 关闭。第二次关闭的影响，按具体的实现定义。
    Close() error
}

type Seeker interface {
    // 设置读/写位置
    Seek(offset int64, whence int) (int64, error)
}

type ReadWriter interface {
    Reader
    Writer
}

type ReadCloser interface {
    Reader
    Closer
}

type WriteCloser interface {
    Writer
    Closer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

type ReadSeeker interface {
    Reader
    Seeker
}

type WriteSeeker interface {
    Writer
    Seeker
}

type ReadWriteSeeker interface {
    Reader
    Writer
    Seeker
}

type ReaderFrom interface {
    // 读取数据直到EOF或出错
    ReadFrom(r Reader) (n int64, err error)
}

type WriterTo interface {
    // 写入数据知道EOF或出错
    WriteTo(w Writer) (n int64, err error)
}

type ReaderAt interface {
    // 从off开始读取最多len(p)个字节。如果读取字节数量小于len(p)，会返回错误。
    // 
    // 可能返回 (len(p), nil)或(len(p), EOF)
    //
    // ReadAt不受Seek的影响。
    //
    // 可以并发读取数据。
    ReadAt(p []byte, off int64) (n int, err error)
}

type WriterAt interface {
    // 从off开始写入字节。如果写入数量小于len(p)，会返回错误。
    // 
    // WriteAt不受Seek的影响。
    //
    // 可以并发写数据。
    WriteAt(p []byte, off int64) (n int, err error)
}

type ByteReader interface {
    // 读取一个字节
    ReadByte() (byte, error)
}

type ByteScanner interface {
    ByteReader
    // 将之前读取的字节放回
    UnreadByte() error
}

type ByteWriter interface {
    // 写入一个字节
    WriteByte(c byte) error
}

type RuneReader interface {
    // 读取一个UTF-8字符
    ReadRune() (r rune, size int, err error)
}

type RuneScanner interface {
    RuneReader
    // 将之前读取的UTF-8字符放回
    UnreadRune() error
}

type StringWriter interface {
    // 写入字符串
    WriteString(s string) (n int, err error)
}
```

## 结构体

```go
type LimitedReader struct {
    R Reader // underlying reader
    N int64  // max bytes remaining
}

type SectionReader struct {
    r     ReaderAt
    base  int64
    off   int64
    limit int64
}

type teeReader struct {
    r Reader
    w Writer
}
```

## 函数

- `WriteString(w Writer, s string) (n int, err error)`

写入字符串。

- `ReadAtLeast(r Reader, buf []byte, min int) (n int, err error)`

至少读取`min`字节。
如果len(buf)小于min，返回`ErrShortBuffer`。
如果读取字节不足`min`字节返回`ErrUnexpectedEOF`。
如果无法从`r`读取任何字节，返回`EOF`。
如果已经从`r`读取了`min`字节，即使`r`报错也会被忽略。

- `ReadFull(r Reader, buf []byte) (n int, err error)`

读取len(buf)字节。
如果读取字节不足len(buf)字节返回`ErrUnexpectedEOF`。
如果无法从`r`读取任何字节，返回`EOF`。
如果已经从`r`读取了len(buf)字节，即使`r`报错也会被忽略。

- `CopyN(dst Writer, src Reader, n int64) (written int64, err error)`

从src拷贝n字节到dst。如果不足会返回EOF。

- `Copy(dst Writer, src Reader) (written int64, err error)`

从src拷贝n字节到dst。重复调用，也不会返回EOF错误。

- `CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error)`

使用缓冲区buf，从src拷贝n字节到dst。重复调用，也不会返回EOF错误。

- `LimitReader(r Reader, n int64) Reader`

创建LimitReader。该Reader读完`n`字节后会返回`EOF`。

- `NewSectionReader(r ReaderAt, off int64, n int64) *SectionReader`

创建SectionReader。该Reader，从`off`开始读取，并且读完`n`字节后会返回`EOF`。

- `TeeReader(r Reader, w Writer) Reader`

创建TeeReader，从`r`读取数据时，会同时将读取到的数据写到`w`。

## 源码解析

```go
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
    // 如果src实现了WriteTo，直接使用，避免分配缓冲区并拷贝
    if wt, ok := src.(WriterTo); ok {
        return wt.WriteTo(dst)
    }
    // 如果dst实现了ReadFrom，直接使用，避免分配缓冲区及拷贝
    if rt, ok := dst.(ReaderFrom); ok {
        return rt.ReadFrom(src)
    }
    if buf == nil {
        size := 32 * 1024
        if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
            // 如果srs为LimitedReader类型，可以缩小缓冲区大小
            if l.N < 1 {
                size = 1
            } else {
                size = int(l.N)
            }
        }
        // 分配缓冲区
        buf = make([]byte, size)
    }
    for {
        nr, er := src.Read(buf) // 读取数据到缓冲区
        if nr > 0 { 
            // 先处理读取到的数据，这里先不管是否发生错误（即使发生错误，也需返回写入多少字节）
            nw, ew := dst.Write(buf[0:nr])
            if nw > 0 {
                written += int64(nw)
            }
            if ew != nil {
                // 写入错误
                err = ew
                break
            }
            if nr != nw {
                // 写入数据量不足
                err = ErrShortWrite
                break
            }
        }
        if er != nil { 
            // 处理错误情况
            if er != EOF {
                err = er
            }
            break
        }
    }
    return written, err
}
```