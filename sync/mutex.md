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
	state int32
	sema  uint32
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
        // 饥饿模式不自旋，也无法获得锁。唤醒模式也不自旋
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 尝试设置锁状态为唤醒，但不唤醒其他协程
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
        // 饥饿状态的锁，不要尝试获取锁，新的协程必须进入队列
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
        // 当前协程将锁切换到饥饿模式。
        // 但如果已经解锁，不用切换，Unlock假设饥饿模式都有等待者。
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// If we were already waiting before, queue at the front of the queue.
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
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
		old := new
		for {
			// If there are no waiters or a goroutine has already
			// been woken or grabbed the lock, no need to wake anyone.
			// In starvation mode ownership is directly handed off from unlocking
			// goroutine to the next waiter. We are not part of this chain,
			// since we did not observe mutexStarving when we unlocked the mutex above.
			// So get off the way.
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// Grab the right to wake someone.
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
        // 饥饿模式：
        // 将归属权移交给下一个等待者，使下一个等待者可以立即运行。
        // 如果mutexStarving被设置，互斥锁任然会被任务是锁住的，新的等待者将不能获得锁。
		runtime_Semrelease(&m.sema, true, 1)
	}
}

```