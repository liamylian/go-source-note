# 互斥锁

## 常量

```go
const(
	mutexLocked         = 1 // 已上锁
	mutexWoken          = 2 // 唤醒
	mutexStarving       = 4 // 
	mutexWaiterShift    = 3 // 唤醒中

    // 公平唤醒机制
    // 互斥锁有两种模式：普通及饥饿模式。普通模式下，等待者以先进先出的队列排列。
    // 但是唤醒的等待者，需要与新的等待者竞争，并且新的等待者有更大的竞争力（正在拥有CPU，并且数量多）。
    // 故如果等待者超过1ms仍未得到锁，将进入饥饿模式。
	starvationThresholdNs = 1e6
)
```

## 类型

```go
// 锁接口
type Locker interface {
	Lock()
	Unlock()
}

// 使用后不能复制
type Mutex struct {
	state int32 // 状态标识，第一位标识是否锁住，第2位标识是否被唤醒，第三位是否饥饿，第4~32位标识等待mutex上的协程数量。
	sema  uint32 // 信号量
}
```

## Mutex方法

- `Lock()`

加锁。阻塞知道获取到锁。

- `Unlock()`

解锁。如果解锁前未加锁将奔溃。

## 源码解析

```go
func (m *Mutex) Lock() {
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        // 如果比较替换成功，即未上锁过，可快速上锁
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
    // 阻塞直到获得锁
	m.lockSlow()
}

func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
        // 饥饿模式不自旋，也无法获得锁。
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 如果没有唤醒的协程，当前协程未唤醒，而且有等待的协程，尝试设置锁状态为唤醒
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
            // 进入自旋
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
		new := old
		if old&mutexStarving == 0 {
            // 不是饥饿模式，标识上锁
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
            // 增加等待协程数
			new += 1 << mutexWaiterShift
		}

		if starving && old&mutexLocked != 0 {
            // 切换到饥饿模式
			new |= mutexStarving
		}
		if awoke {
            // 协程已唤醒，检查状态是否一致
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
            // 取消唤醒标识
			new &^= mutexWoken 
		}
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
                // 未被锁住，可以返回了
				break 
			}
            // 如果等待过，放入后进先出队列
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
            // 如果等待时间超过阀值，设置为饥饿模式
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    // 状态不一致：饥饿模式，且被唤醒或加锁
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
                    // 退出饥饿模式
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}

func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
        // 如果不为0，说明处在其他状态
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
        // 非饥饿模式
		old := new
		for {
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				// 没有等待的协程，或者状态在加锁、唤醒、饥饿模式，返回
				return
			}
            // 设置为唤醒，并且减少等待者
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
                // 释放信号量
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
        // 饥饿模式：
        // 将归属权移交给下一个等待者，使下一个等待者可以立即运行。
		runtime_Semrelease(&m.sema, true, 1)
	}
}

```