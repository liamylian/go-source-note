# 缓冲区

## 常量

```go
// 读操作
type readOp int8

const (
    opRead      readOp = -1 // 其他读操作
    opInvalid   readOp = 0  // 非读操作
    opReadRune1 readOp = 1  // 读1字节的`rune`
    opReadRune2 readOp = 2  // 读2节的`rune`
    opReadRune3 readOp = 3  // 读3字节的`rune`
    opReadRune4 readOp = 4  // 读3字节的`rune`
)
```

## 类型

```go
type Buffer struct {
    buf      []byte // 数据为buf[off : len(buf)]
    off      int    // 从&buf[off]开始读，从&buff[len(buf)]开始写
    lastRead readOp // 最后一次读操作，这样下一次`Unread`才能实现.
}
```

## 函数

### `Buffer`的方法

- `Bytes() []byte`

返回未读取字节切片。这个切片需在下一次修改缓冲区（如Read, Write, Reset, or Truncate）前使用。
修改该切片也会影响缓冲区的后续读取操作。

- `String() string`

以字符串格式返回未读取的数据。

- `Len() int`

返回未读取字节长度。

- `Cap() int`

返回缓冲区容量。

- `Truncate(n int)`

写n个空字节。

- `Reset()`

重置为空缓冲区。

- `Grow(n int)`

增加缓冲区容量，使之至少保证能继续写入`n`字节。

- `Write(p []byte) (n int, err error)`

写入字节。

- `WriteString(s string) (n int, err error)`

写入字符串。

- `ReadFrom(r io.Reader) (n int64, err error)`

从`r`读取数据，并写入缓冲区。

- `WriteTo(w io.Writer) (n int64, err error)`

从缓冲区读取数据，并写入到`w`。

- `WriteByte(c byte) error`

写入一个字节。

- `WriteRune(r rune) (n int, err error)`

写入一个rune字符。

- `Read(p []byte) (n int, err error) `

读取字节，并放入`p`。

- `Next(n int) []byte`

读取n个字节。

- `ReadByte() (byte, error)`

读取一个字节。

- `ReadRune() (r rune, size int, err error)`

读取一个`rune`字符。

- `UnreadRune() error`

取消读取一个`rune`字符。

- `UnreadByte() error`

取消读取一个字节。

- `ReadBytes(delim byte) (line []byte, err error)`

读取字节直到`delim`。

- `ReadString(delim byte) (line string, err error)`

读取字符串直到`delim`。

## 源码详解

```go
// 切片还有容量，通过重新切片扩容。
func (b *Buffer) tryGrowByReslice(n int) (int, bool) {
    if l := len(b.buf); n <= cap(b.buf)-l {
        b.buf = b.buf[:l+n]
        return l, true
    }
    return 0, false
}

// 增加缓冲区容量，使之至少保证能继续写入`n`字节。返回下一次用于写入的下标。
func (b *Buffer) grow(n int) int {
    m := b.Len()
    // 没有空间，先重置
    if m == 0 && b.off != 0 {
        b.Reset()
    }
    
    // 尝试使用重新切片扩容
    if i, ok := b.tryGrowByReslice(n); ok {
        return i
    }
    
    // 缓冲区为空，并且n的大小小于小缓冲区大小，创建小缓冲区
    if b.buf == nil && n <= smallBufferSize {
        b.buf = make([]byte, n, smallBufferSize)
        return 0
    }
    
    c := cap(b.buf)
    if n <= c/2-m { // (n + m) == c/2; n + m <= 2*c +n; c/2 <= 2c +n
        // 如果 扩容字节数+切片长度 < 切片容量/2，不需要扩容。移动一下即可。
        copy(b.buf, b.buf[b.off:])
    } else if c > maxInt-c-n {
        // 无法扩容到2*c+n
        panic(ErrTooLarge)
    } else {
        // 容量不足，必须重新创建切片。并且新切片容量为旧切片容量的2倍加需扩容字符数。
        buf := makeSlice(2*c + n)
        copy(buf, b.buf[b.off:])
        b.buf = buf
    }
    
    // 重置缓冲区
    b.off = 0
    b.buf = b.buf[:m+n]
    return m
}

// 增加缓冲区容量，使之至少保证能继续写入`n`字节。
func (b *Buffer) Grow(n int) {
    if n < 0 {
        panic("bytes.Buffer.Grow: negative count")
    }
    m := b.grow(n)
    b.buf = b.buf[:m]
}
```