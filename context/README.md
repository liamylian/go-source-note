# 上下文

## 依赖类型

- `net.Error`

```go
// An Error represents a network error.
type Error interface {
    error
    Timeout() bool   // Is the error a timeout?
    Temporary() bool // Is the error temporary?
}

```
## 变量

```go
var (
    Canceled            = errors.New("context canceled")
    DeadlineExceeded    = deadlineExceededError{} // deadlineExceededError实现了`net.Error`
    background          = new(emptyCtx) // emptyCtx实现了`Context`
    todo                = new(emptyCtx)
)
```
## 类型

```go
// 上下文
type Context interface {
    // 上下文超时信息。
    // 
    // 上下文未设置结束时间时返回`ok`=false
    Deadline() (deadline time.Time, ok bool)

    // 上下文管道（用于标识是否结束）。
    //
    // 如果该上下文不可被取消，返回`nil`
    Done() <-chan struct{}

    // 上下文结束原因
    //
    // 如果上下文未结束，返回nil。
    // 如果上下文结束，返回非空error（`Canceled`/`DeadlineExceeded`）
    Err() error

    // 获取上下文变量（使用`context.WithValue`带上）。
    Value(key interface{}) interface{}
}

// 上下文取消函数
// 
// 调用`CancelFunc`将取消所有子上下文，将自己从父上下文中移除，并停止关联计时器。
// 取消失败将导致子上下午泄漏，指导父上下文或计时器到期。
type CancelFunc func()

```
## 函数

- `Backgroud() Context`

返回不可取消上下文`background`

- `TODO() Context`

返回不可取消上下文`todo`

- `WithCancel(parent Context) (ctx Context, cancel CancelFunc)`

复制父上下，并生成一个可取消的上下文。该上下文当父上下文结束，或`CancelFunc`被执行时被取消。

- `WithDeadline(parent Context, d time.Time) (Context, CancelFunc)`

复制父上下，并生成一个可带截止时间的上下文。该上下文当父上下文结束，`CancelFunc`被执行或到期时被取消。

- `WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)`

复制父上下，并生成一个可带超时时间的上下文。该上下文当父上下文结束，或`CancelFunc`被执行或超时时被取消。

- `WithValue(parent Context, key, val interface{}) Context`

复制父上下，并生成一个可带参数的上下文。

`key`参数不应当使用任何基本类型，以避免冲突。而应当使用用户自己的类型。

## 详解

### `WithCancel`上下文实现

```go
// 绑定子上下文的取消到父上下文
func propagateCancel(parent Context, child canceler) {
    done := parent.Done()
    if done == nil {
        return // 父上下文永远不会被取消，无需绑定
    }

    select {
    case <-done:
        // 父上下文已经取消，无需绑定
        child.cancel(false, parent.Err())
        return
    default:
    }

    if p, ok := parentCancelCtx(parent); ok {
        // 获取到父上下文的`*cancelCtx`。 通过给父上下文添加子上下文的方式，实现绑定。
        // `*cancelCtx`在关闭时，会同时取消子上下文。
        p.mu.Lock()
        if p.err != nil {
            // 父上下文已经取消，取消子上下文
            child.cancel(false, p.err)
        } else {
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
    } else {
        // 如果父上下文无`*cancelCtx`，使用协程方式实现绑定监听`
        // 当父上下文被取消后，取消子上下文。
        atomic.AddInt32(&goroutines, +1)
        go func() {
            select {
            case <-parent.Done():
                child.cancel(false, parent.Err())
            case <-child.Done():
            }
        }()
    }
}

// 获取父上下文的`*cancelCtx`
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    done := parent.Done()
    if done == closedchan || done == nil {
        // 如果已经取消，或者永远不会被取消，返回错误。
        return nil, false
    }

    p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
    if !ok {
        // 如果不存在`*cancelCtx`返回错误
        return nil, false
    }

    p.mu.Lock()
    ok = p.done == done // 判断parent.Done()是否被封装过。
    p.mu.Unlock()
    if !ok {
        // 如果`parent.Done()`已经被封装过（使用不同的`done`管道），返回错误
        return nil, false
    }
    return p, true
}

// 将子上下文从从父上下文移除
func removeChild(parent Context, child canceler) {
    p, ok := parentCancelCtx(parent)
    if !ok {
        return
    }
    p.mu.Lock()
    if p.children != nil {
        delete(p.children, child)
    }
    p.mu.Unlock()
}

// 当自身被取消，将同时取消所有子上下文
type cancelCtx struct {
    Context

    mu       sync.Mutex            // 保护以下字段
    done     chan struct{}         // 延迟赋值，由第一次取消关闭
    children map[canceler]struct{} // 第一次取消后，设置为nil
    err      error                 // 第一次取消后，设置为非空
}

// 关闭`c.done`，取消所有子上下文，而且如果`removeFromParent`为真，将自己从父上下文移除。
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // 已经被取消
    }

    // 设置`c.err`及`c.done`
    c.err = err 
    if c.done == nil {
        c.done = closedchan
    } else {
        close(c.done)
    }

    // 取消所有子上下文
    for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        // 将自身从父上下文移除
        removeChild(c.Context, c)
    }
}
```

## `WithDeadline`上下文实现

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // 已经到期
        return WithCancel(parent)
    }
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }

    // 绑定到父上下文
    propagateCancel(parent, c)
    dur := time.Until(d)
    if dur <= 0 {
        // 已过期，取消
        c.cancel(true, DeadlineExceeded) 
        return c, func() { c.cancel(false, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        // 使用定时器定时取消
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}

type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.

    deadline time.Time
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
    // 取消父上下文
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        // 将自己从父上下文的子上下文列表中移除
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        // 关闭定时器
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

## `WithValue`上下文实现

```go
func WithValue(parent Context, key, val interface{}) Context {
    if key == nil {
        panic("nil key")
    }
    if !reflectlite.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}
}

type valueCtx struct {
    Context
    key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        // 如果`key`相等，返回上下文
        return c.val
    }
    // 尝试从父上下文获取
    return c.Context.Value(key)
}
```