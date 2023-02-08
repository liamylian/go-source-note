# 系统日志

简单系统日志实现。通过UNIX域套接字，UDP或TCP的方式将日志发送给`系统日志服务`。
仅需一次手动连接，当写入失败时，实现会自动重连。

## 常量

- 日志级别

|名称|数值|描述|
|---|---|---|
|LOG_EMERG|0|紧急情况，如即将发生的系统崩溃，通常会广播到所有用户|
|LOG_ALERT|1|需要立即修改的情况，如系统数据库的损坏|
|LOG_CRIT|2|关键的情况，如一个硬件的错误|
|LOG_ERR|3|普通错误|
|LOG_WARNING|4|警告|
|LOG_NOTICE|5|不是一个错误的情况，但是可能需要用特定方式的处理一下|
|LOG_INFO|6|报告性的消息|
|LOG_DEBUG|7|用于调试程序的消息|

- 日志事件类型

|名称|数值|描述|
|---|---|---|
|LOG_KERN|0|内核产生的消息|
|LOG_USER|1<<3|随机用户进程产生的消息|
|LOG_MAIL|1<<4|电子邮件系统|
|LOG_DAEMON|1<<5|系统守护进程（除cron）|
|LOG_AUTH|1<<6|认证系统：login、su、getty等|
|LOG_SYSLOG|1<<7|自身产生的日志|
|LOG_LPR|1<<8|系统打印机缓冲池：lpr、lpd|
|LOG_NEWS|1<<9|网络新闻系统|
|LOG_UUCP|1<<10|UUCP子系统|
|LOG_CRON|1<<11|cron守护进程|
|LOG_AUTHPRIV|1<<12|同LOG_AUTH，但只登录到所选择的单个用户可读的文件中|
|LOG_FTP|1<<13|文件传输协议：ftpd、tftpd|
||1<<14|未使用|
||1<<15|未使用|
||1<<16|未使用|
||1<<17|未使用|
|LOG_LOCAL0|1<<18|用户自定义设备0|
|LOG_LOCAL1|1<<19|用户自定义设备1|
|LOG_LOCAL2|1<<20|用户自定义设备2|
|LOG_LOCAL3|1<<21|用户自定义设备3|
|LOG_LOCAL4|1<<22|用户自定义设备4|
|LOG_LOCAL5|1<<23|用户自定义设备5|
|LOG_LOCAL6|1<<24|用户自定义设备6|
|LOG_LOCAL7|1<<25|用户自定义设备7|

## 类型

```go
// 系统日志客户断
type Writer struct {
    priority Priority // 级别
    tag      string   // 前缀
    hostname string   // 主机名
    network  string   // 网络
    raddr    string   // 网路地址

    mu   sync.Mutex   // 保护下面字段
    conn serverConn   // 网络连接
}

// 仅用于对Solaris系统进行支持
// Solaris系统上不能通过TCP方式连接到系统日志守护进程，而是需要通过库函数方式进行写日志。
type serverConn interface {
    writeString(p Priority, hostname, tag, s, nl string) error
    close() error
}
```

## 函数

- `New(priority Priority, tag string) (*Writer, error)`

建立系统日志服务客户端。该连接的所有日志消息，都会带上日志级别`priority`及前缀`tag`。

- `Dial(network, raddr string, priority Priority, tag string) (*Writer, error)`

建立系统日志服务客户端。该连接的所有日志消息，都会带上日志级别`priority`及前缀`tag`。
如果`tag`为空，则使用`os.Args[0]`。如果`network`为空，默认连接到本地系统日志服务，否则按`network`及`raddr`进行连接。

- `NewLogger(p Priority, logFlag int) (*log.Logger, error)`

创建日志服务。该日志服务将消息写到系统日志服务，并且所有日志消息，都会带上日志级别`priority`。
`logFlag`为创建日志服务时传递的`flag`属性参数。

### `Writter`方法

- `Write(b []byte) (int, error)`

发送日志。

- `Close() error`

关闭连接。

- `Emerg(m string) error`

发送紧急级别日志，忽略创建连接时的日志级别。

- `Alert(m string) error`

发送告警级别日志，忽略创建连接时的日志级别。

- `Crit(m string) error`

发送关键级别日志，忽略创建连接时的日志级别。

- `Err(m string) error`

发送错误级别日志，忽略创建连接时的日志级别。

- `Warning(m string) error`

发送警告级别日志，忽略创建连接时的日志级别。

- `Notice(m string) error`

发送提示级别日志，忽略创建连接时的日志级别。

- `Info(m string) error`

发送报告级别日志，忽略创建连接时的日志级别。

- `Debug(m string) error`

发送调试级别日志，忽略创建连接时的日志级别。

## 源码解读

- 建立系统日志服务连接，及失败重连

```go
// 使用域套接字方式，连接到系统日志服务
func unixSyslog() (conn serverConn, err error) {
    logTypes := []string{"unixgram", "unix"}
    logPaths := []string{"/dev/log", "/var/run/syslog", "/var/run/log"}
    for _, network := range logTypes {
        for _, path := range logPaths {
            conn, err := net.Dial(network, path)
            if err == nil {
                return &netConn{conn: conn, local: true}, nil
            }
        }
    }
    return nil, errors.New("Unix syslog delivery error")
}

// 连接系统日志服务
func (w *Writer) connect() (err error) {
    if w.conn != nil {
        // 先关闭旧连接，并忽略错误
        w.conn.close()
        w.conn = nil
    }

    if w.network == "" {
        // 连接到本地系统日志服务（使用Unix域套接字）
        w.conn, err = unixSyslog()
        if w.hostname == "" {
            w.hostname = "localhost"
        }
    } else {
        // 连接到指定的系统日志服务
        var c net.Conn
        c, err = net.Dial(w.network, w.raddr)
        if err == nil {
            w.conn = &netConn{conn: c}
            if w.hostname == "" {
                w.hostname = c.LocalAddr().String()
            }
        }
    }
    return
}

func (w *Writer) writeAndRetry(p Priority, s string) (int, error) {
    // 重新计算日志级别（保留初始指定的日志设施，并使用当前指定的日志级别）
    pr := (w.priority & facilityMask) | (p & severityMask)

    w.mu.Lock()
    defer w.mu.Unlock()

    if w.conn != nil {
        if n, err := w.write(pr, s); err == nil {
        // 写入成功退出
            return n, err
        }
    }
    
    // 连接为空或者写入失败，重新连接
    if err := w.connect(); err != nil {
        return 0, err
    }
    return w.write(pr, s)
}
```

- 格式化输出

```go
// 写日志。
func (w *Writer) write(p Priority, msg string) (int, error) {
    // 如果没有换行则追加换行
    nl := ""
    if !strings.HasSuffix(msg, "\n") {
        nl = "\n"
    }

    err := w.conn.writeString(p, w.hostname, w.tag, msg, nl)
    if err != nil {
        return 0, err
    }
    // 返回`msg`长度，而非格式化后的完整日志长度。为了保证与`io.Writer`行为一致。
    return len(msg), nil
}

// 以`<PRI>TIMESTAMP HOSTNAME TAG[PID]: MSG`格式输出日志
func (n *netConn) writeString(p Priority, hostname, tag, msg, nl string) error {
    if n.local {
        // 若为本机网络，使用time.Stamp格式输出时间，并且去除`hostname`字段
        timestamp := time.Now().Format(time.Stamp)
        _, err := fmt.Fprintf(n.conn, "<%d>%s %s[%d]: %s%s",
            p, timestamp,
            tag, os.Getpid(), msg, nl)
        return err
    }
    timestamp := time.Now().Format(time.RFC3339)
    _, err := fmt.Fprintf(n.conn, "<%d>%s %s %s[%d]: %s%s",
        p, timestamp, hostname,
        tag, os.Getpid(), msg, nl)
    return err
}
```