# once

## 类型

```go
type Once struct {
	done uint32
	m    Mutex
}
```

## Once方法

- `Do(f func())`

只执行一次`f`函数。

## 源码解析

```go
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		// done为0，表示还为执行过
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock() // 加锁，防止并发进入
	defer o.m.Unlock()
	if o.done == 0 { // 再次检查done是否为0
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```