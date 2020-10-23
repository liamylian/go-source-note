# cond

## 类型

```go
type Cond struct {
	noCopy noCopy

	// L is held while observing or changing the condition
	L Locker

	notify  notifyList
	checker copyChecker
}
```

## 函数

- `NewCond(l Locker) *Cond`

创建条件变量

## `Cond`方法

- `Wait()`

解锁`c.L`并阻塞。当恢复执行时，对`c.L`进行上锁。

```go
// 使用方式
c.L.Lock() // 准备借箭
for !condition() {
    c.Wait() // 东风未到，继续等待
}

// 东风到了，可以借箭了
c.L.Unlock() // 完成借箭
```

- `Signal()`

唤醒一个协程。

- `Broadcast()`

唤醒所有协程。

## 源码解析

```go
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}

// Signal wakes one goroutine waiting on c, if there is any.
//
// It is allowed but not required for the caller to hold c.L
// during the call.
func (c *Cond) Signal() {
	c.checker.check()
	runtime_notifyListNotifyOne(&c.notify)
}

// Broadcast wakes all goroutines waiting on c.
//
// It is allowed but not required for the caller to hold c.L
// during the call.
func (c *Cond) Broadcast() {
	c.checker.check()
	runtime_notifyListNotifyAll(&c.notify)
}


// 检查是否被复制
type copyChecker uintptr

func (c *copyChecker) check() {
	if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
		!atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
		uintptr(*c) != uintptr(unsafe.Pointer(c)) {
        // 如果下面条件满足，表示被复制了
        // 1. c的值与c的指针不一致
        // 2. 比较c的值是否为0，并且替换为新值失败（第一次check，会替换成功，并保存自己的指针地址）
        // 3. c的值与c的指针还是不一致（复制后，可以指针指向了复制前的结构体）
		panic("sync.Cond is copied")
	}
}
```