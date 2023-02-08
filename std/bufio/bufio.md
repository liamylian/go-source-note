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

读取字节。

- `ReadByte() (byte, error)`

读取一个字节。

- `UnreadByte() error`

放回读取的一个字节。

- `ReadRune() (r rune, size int, err error)`

读取一个UTF-8字符

- `UnreadRune() erro`

放回读取的一个UTF-8字符

- `Buffered() int`

缓冲的字节数。

- `ReadSlice(delim byte) (line []byte, err error)`

读取字节串直到分隔符。如果没有读取到分隔符，会返回错误（通常为io.EOF）
如果缓冲区内没有分隔符，返回ErrBufferFull。
返回的切片，在下次读取后将实效，建议使用ReadBytes和ReadString。

- `ReadLine() (line []byte, isPrefix bool, err error)`

读取一行数据，并且不包括换行符，如果行过长会截断，并且isPrefix为`true`。
返回的切片，在下次读取后将实效，建议使用ReadBytes('\n')和ReadString('\n')。

- `ReadBytes(delim byte) ([]byte, error)`

读取字节串直到分隔符。如果没有读取到分隔符，会返回错误（通常为io.EOF）
如果缓冲区内没有分隔符，返回ErrBufferFull。

- `ReadString(delim byte) (string, error)`

读取字符串直到分隔符。如果没有读取到分隔符，会返回错误（通常为io.EOF）
如果缓冲区内没有分隔符，返回ErrBufferFull。

- `WriteTo(w io.Writer) (n int64, err error)`

读取所有数据，并写入到w。

## `Writer`方法

- `Size() int`

返回缓冲区大小。

- `Reset(r io.Writer)`

重置缓冲区。

- `Flush() error`

将缓冲区的是哦有数据写出。

- `Available() int`

剩余缓冲区大小。

- `Buffered() int`

缓冲数据量。

- `Write(p []byte) (nn int, err error)`

将数据写入缓冲区，如果写入数据不足len(p)，会返回错误。

- `WriteByte(c byte) error`

写入一个字符。

- `WriteRune(r rune) (size int, err error)`

写入一个UTF-8字符。

- `WriteString(s string) (int, error)`

写入字符串。

- `ReadFrom(r io.Reader) (n int64, err error)`

从读取器读取数据，并写入缓冲区。