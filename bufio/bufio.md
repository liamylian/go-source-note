# Bufio

## 变量

```go
var (
	ErrInvalidUnreadByte = errors.New("bufio: invalid use of UnreadByte")
	ErrInvalidUnreadRune = errors.New("bufio: invalid use of UnreadRune")
	ErrBufferFull        = errors.New("bufio: buffer full")
	ErrNegativeCount     = errors.New("bufio: negative count")
)
```

## 类型

```go
type Reader struct {
	buf          []byte
	rd           io.Reader  // 客户端提供的Reader
	r, w         int        // 缓冲区读写位置
	err          error
	lastByte     int        // 用于UnreadByte的最后一次读取字节; -1为无效
	lastRuneSize int        // 用于UnreadByte的最后一次读取字符; -1为无效
}

type Writer struct {
	err error
	buf []byte
	n   int
	wr  io.Writer
}
```

## 函数

- `NewReaderSize(rd io.Reader, size int) *Reader`

创建读取缓冲器，缓冲区大小至少为size字节。如过原来的读取器`rd`缓冲区大小充足，则直接返回`rd`。

- `NewReader(rd io.Reader) *Reader`

创建读取缓冲器，缓冲区大小使用默认配置。

- `NewWriterSize(w io.Writer, size int) *Writer`

创建写缓冲器，缓冲区大小至少为size字节。如过原来的写缓冲器`rd`缓冲区大小充足，则直接返回`rd`。

- `NewWriter(w io.Writer) *Writer`

创建写缓冲器，缓冲区大小使用默认配置。

- `NewReadWriter(r *Reader, w *Writer) *ReadWriter`

创建读写缓冲器。

## `Reader`方法

- `Size() int`

返回缓冲区大小。

- `Reset(r io.Reader)`

重置缓冲区。

- `Peek(n int) ([]byte, error)`

探取`n`个 字节，如果字节不足会返回错误。如果n过大，会返回`ErrBufferFull`。
Peek会导致，下次`UnreadByte`和`UnreadRune`调用失败。

- `Discard(n int) (discarded int, err error)`

丢弃`n`个字节。如果丢弃的字节数少于`n`个字节，返回具体错误。

- `Read(p []byte) (n int, err error)`

- `ReadByte() (byte, error)`

- `UnreadByte() error`

- `ReadRune() (r rune, size int, err error)`

- `UnreadRune() erro`

- `Buffered() int`

- `ReadSlice(delim byte) (line []byte, err error)`

- `ReadLine() (line []byte, isPrefix bool, err error)`

- `ReadBytes(delim byte) ([]byte, error)`

- `ReadString(delim byte) (string, error)`

- `WriteTo(w io.Writer) (n int64, err error)`


## `Writer`方法

- `Size() int`

- `Reset(w io.Writer)`

- `Flush() error`

- `Available() int`

- `Buffered() int`

- `Write(p []byte) (nn int, err error)`

- `WriteByte(c byte) error`

- `WriteRune(r rune) (size int, err error)`

- `WriteString(s string) (int, error)`

- `ReadFrom(r io.Reader) (n int64, err error)`
