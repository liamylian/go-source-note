# WaitGroup

## 类型

```go
type WaitGroup struct {
	noCopy noCopy

    // 64位整型：高32位做为计数，低32位等待计数
    // 32位整型：信号量
	state1 [3]uint32
}
```

## WaitGroup方法

- `Add(delta int)`

增加计数。

- `Done()`

减少计数。

- `Wait()`

阻塞，直到计数为0。

## 源码解析

```go
// 返回计数器及信号量
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}

// 1. Add与Wait不能并发调用
// 2. 如果重复利用WaitGroup，必须要等待所有Wait返回后再调用Add
func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()
	if race.Enabled {
		_ = *statep
		if delta < 0 {
			race.ReleaseMerge(unsafe.Pointer(wg))
		}
		race.Disable()
		defer race.Enable()
	}

	state := atomic.AddUint64(statep, uint64(delta)<<32) // 增加计数
	v := int32(state >> 32)
	w := uint32(state)
	if race.Enabled && delta > 0 && v == int32(delta) {
		race.Read(unsafe.Pointer(semap))
	}
	if v < 0 {
        // 计数为负数
		panic("sync: negative WaitGroup counter")
	}
	if w != 0 && delta > 0 && v == int32(delta) {
        // `Add`与`Wait`并发调用
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	if v > 0 || w == 0 {
        // 还没有调用Wait，或者计数大于0
		return
	}
	if *statep != state {
        // 发现`Add`与`Wait`并发调用，否则值应该一致
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}

	// 引用计数已减到0，并且有等待计数
	*statep = 0 // 重置计数为0
	for ; w != 0; w-- {
        // 增加*semap，并通知阻塞的协程
		runtime_Semrelease(semap, false, 0)
	}
}

func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()
	if race.Enabled {
		_ = *statep / trigger nil deref early
		race.Disable()
	}
	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32)
		w := uint32(state)
		if v == 0 {
			// 没有计数，直接返回
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
		// 增加等待计数
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			if race.Enabled && w == 0 {
				race.Write(unsafe.Pointer(semap))
			}
			runtime_Semacquire(semap) // 阻塞直到*semap>0, 然后减少*semap
			if *statep != 0 {
                // Wait还未结束，就进行Add
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
	}
}


```