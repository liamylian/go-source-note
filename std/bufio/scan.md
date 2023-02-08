# Scan

## 变量

```go
var (
	ErrTooLong         = errors.New("bufio.Scanner: token too long")
	ErrNegativeAdvance = errors.New("bufio.Scanner: SplitFunc returns negative advance count")
	ErrAdvanceTooFar   = errors.New("bufio.Scanner: SplitFunc returns advance count beyond input")

    ErrFinalToken = errors.New("final token") // 最后一个令牌。用于提前停止。
)
```

## 类型

```go
// atEOF 表示没有数据了
// 返回的 advance表示需要向前移动多少字节，token为扫描结果。
// 
// 如果返回错误，则停止扫描。
// 如果返回的结果非空，将结果返回给用户。如果结果为空，则需要重试扫描直到EOF。
type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)
```

## 函数

- `NewScanner(r io.Reader) *Scanner`

创建扫描器，默认扫描器为行扫描。

- `ScanBytes(data []byte, atEOF bool) (advance int, token []byte, err error)`

字节扫描分割函数。

- `ScanRunes(data []byte, atEOF bool) (advance int, token []byte, err error)`

UTF8字符扫描分割函数。

- `ScanLines(data []byte, atEOF bool) (advance int, token []byte, err error)`

行扫描分割函数。

- `ScanWords(data []byte, atEOF bool) (advance int, token []byte, err error)`

词扫描分割函数。

## Scanner方法

- `Err() error`

返回第一个非EOF错误。

- `Bytes() []byte`

返回最近一次扫描结果，该结果可能会被下次扫描覆盖。

- `Text() string`

返回最近一次扫描字符串结果，该结果可能会被下次扫描覆盖。

- `Scan() bool`

扫描数据。如果返回false，表示结束。

- `Buffer(buf []byte, max int)`

设置缓冲区。

- `Split(split SplitFunc)`

设置分割函数。
