# 日志

所有日志消息都会在单独一行，如果日志消息没有换行符，那么会被加上一个换行符。

## 常量

- 日志属性

|常量|数值|描述|
|---|---|---|
|Ldate|1|日期|
|Ltime|2|时间|
|Lmicroseconds|4|微秒（Ltime存在下）|
|Llongfile|8|详细文件及代码位置|
|Lshortfile|16|文件及代码位置|
|LUTC|32|使用UTC时间（当Late, Ltime存在下）|
|Lmsgprefix|64|将前缀从行首移动到消息前|
|LstdFlags|1+2|默认前缀|

## 变量

- 默认日志输出对象。

```go
var std = New(os.Stderr, "", LstdFlags)
```

## 类型

- `Logger`

## 函数

- `New(out io.Writer, prefix string, flag int) *Logger`

创建日志对象。如果`flag`设置了`Lmsgprefix`，`prefix`不是放在行首，而是放在消息前。

- `SetOutput(w io.Writer)`

设置日志输出位置。

- `Flags() int`

获取日志属性。

- `SetFlags(flag int)`

设置日志属性。

- `Prefix() string`

获取日志输出前缀。

- `SetPrefix(prefix string)`

设置日志输出前缀。

- `Writer() io.Writer`

获取日志输出位置。

- `Print(v ...interface{})`

输出日志。

- `Printf(format string, v ...interface{})`

格式化输出日志。

- `Println(v ...interface{}) `

输出日志。

- `Fatal(v ...interface{})`

输出日志，并且调用`os.Exit(1)`。

- `Fatalf(format string, v ...interface{})`

格式化输出日志，并且调用`os.Exit(1)`。

- `Fatalln(v ...interface{})`

输出日志，并且调用`os.Exit(1)`。

- `Panic(v ...interface{})`

输出日志，并且调用`panic()`。

- `Panicf(format string, v ...interface{})`

格式化输出日志，并且调用`panic()`。

- `Panicln(v ...interface{})`

输出日志，并且调用`panic()`。

- `Output(calldepth int, s string) error`

输出日志，并且如果设置了`Lshortfile`或`Llongfile`，同时输出执行位置。

## 详解

```go
type Logger struct {
    mu     sync.Mutex // 保护以下字段
    prefix string     // 前缀
    flag   int        // 属性
    out    io.Writer  // 输出位置
    buf    []byte     // 缓冲区
}

func (l *Logger) Output(calldepth int, s string) error {
    now := time.Now() // 先获取时间
    var file string
    var line int
    l.mu.Lock() // 加琐
    defer l.mu.Unlock()
    if l.flag&(Lshortfile|Llongfile) != 0 {
        // 如果需要获取代码执行位置，先释放锁（已经判断完），获取执行位置，然后再重新上锁
        l.mu.Unlock()
   
        // 获取代码位置
        var ok bool
        _, file, line, ok = runtime.Caller(calldepth)
        if !ok {
            file = "???"
            line = 0
        }
        // 再次加上锁
        l.mu.Lock()
    }
    
    l.buf = l.buf[:0] // 清空缓冲区
    l.formatHeader(&l.buf, now, file, line) // 生成日志头
    l.buf = append(l.buf, s...) // 带上日志消息
    // 如果日志末尾不包含换行，那么加上换行。
    if len(s) == 0 || s[len(s)-1] != '\n' {
        l.buf = append(l.buf, '\n')
    }
    // 输出
    _, err := l.out.Write(l.buf)
    return err
}

// 按以下顺序生成日志头
//  * l.prefix (非空，而且Lmsgprefix未设置)
//  * 日期和时间（如果设置了相应标识）
//  * 代码位置（如果设置了响应标识）
//  * l.prefix (非空，而且设置了Lmsgprefix)
func (l *Logger) formatHeader(buf *[]byte, t time.Time, file string, line int) {
    if l.flag&Lmsgprefix == 0 {
        // 未设置Lmsgprefix，添加l.prefix
        *buf = append(*buf, l.prefix...)
    }
    if l.flag&(Ldate|Ltime|Lmicroseconds) != 0 {
        // 设置了时间标识
        if l.flag&LUTC != 0 {
            // 设置了LUTC，使用UTC时区
            t = t.UTC()
        }
        if l.flag&Ldate != 0 {
            // 设置了日期，添加日期
            year, month, day := t.Date()
            itoa(buf, year, 4)
            *buf = append(*buf, '/')
            itoa(buf, int(month), 2)
            *buf = append(*buf, '/')
            itoa(buf, day, 2)
            *buf = append(*buf, ' ')
        }
        if l.flag&(Ltime|Lmicroseconds) != 0 {
            // 设置了时间或微秒，添加时间和微秒
            hour, min, sec := t.Clock()
            itoa(buf, hour, 2)
            *buf = append(*buf, ':')
            itoa(buf, min, 2)
            *buf = append(*buf, ':')
            itoa(buf, sec, 2)
            if l.flag&Lmicroseconds != 0 {
                *buf = append(*buf, '.')
                itoa(buf, t.Nanosecond()/1e3, 6)
            }
            *buf = append(*buf, ' ')
        }
    }
    if l.flag&(Lshortfile|Llongfile) != 0 {
        // 设置了代码位置，添加代码位置。
        if l.flag&Lshortfile != 0 {
            short := file
            for i := len(file) - 1; i > 0; i-- {
                if file[i] == '/' {
                    short = file[i+1:]
                    break
                }
            }
            file = short
        }
        *buf = append(*buf, file...)
        *buf = append(*buf, ':')
        itoa(buf, line, -1)
        *buf = append(*buf, ": "...)
    }
    if l.flag&Lmsgprefix != 0 {
        // 设置了Lmsgprefix，添加l.prefix
        *buf = append(*buf, l.prefix...)
    }
}
```

