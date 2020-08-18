# 读取器

## 类型

```go
// 实现了io.Reader, io.ReaderAt, io.WriterTo, io.Seeker, io.ByteScanner, and io.RuneScanner
type Reader struct {
	s        []byte
	i        int64 // 当前正在读下标
	prevRune int   // 前一个rune字符
}
```
## 函数

### Reader方法

- `Len() int`

返回未读字节数。

- `Size() int64`

返回所有字节数。

- `Read(b []byte) (n int, err error)`

读取字节。

- `ReadAt(b []byte, off int64) (n int, err error)`

从`off`开始读取字节。

- `ReadByte() (byte, error)`

读取一个字节。

- `UnreadByte() error`

取消读取一个字节。

- `ReadRune() (ch rune, size int, err error)`

读取一个`rune`字符。

- `UnreadRune() error`

取消读取一个`rune`字符。

- `Seek(offset int64, whence int) (int64, error)`

移动到`offset`个字符。

- `WriteTo(w io.Writer) (n int64, err error)`

将数据写到`w`

- `Reset(b []byte)`

移动到第0个字符。

## 源码解析

```go
// 读取一个rune字符
func (r *Reader) ReadRune() (ch rune, size int, err error) {
	if r.i >= int64(len(r.s)) {
		r.prevRune = -1
		return 0, 0, io.EOF
	}
	r.prevRune = int(r.i)
	if c := r.s[r.i]; c < utf8.RuneSelf {
		// 如果字符小于`utf8.RuneSelf，即为单字符`utf-8`字符。直接返回即可
		r.i++
		return rune(c), 1, nil
	}
	// 需要使用解析器解析出一个rune字符
	ch, size = utf8.DecodeRune(r.s[r.i:])
	r.i += int64(size)
	return
}
```
