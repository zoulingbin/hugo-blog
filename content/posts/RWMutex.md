---
title: "RWMutex:读写锁的实现原理"
date: 2021-07-23T02:38:45+08:00
tags: ["Go", "标准库", "并发编程"]
author: zoulingbin
categories: ["Go并发编程"]
description: 鸟窝大佬的go并发编程阅读记录，RWMutex的实现原理

---
<!--more-->

### 什么是 RWMutex？
`mutex`保证只有一个`goroutine`访问共享资源，在读多写少的场景里，使用mutex会造成大量的浪费，大量的读操作在mutex的保护里不能不变成串行读，对性能的影响非常大。而使用`RWMutex`就可以解决此类readers-writers问题。

如果某个读操作的`goroutine`持有了锁，在这种情况下，其它读操作的`goroutine`就不必一直傻傻地等待了，而是可以并发地访问共享变量，这样我们就可以将串行的读变成并行读，提高读操作的性能。当写操作的`goroutine`持有锁的时候，它就是一个排外锁，其它的写操作和读操作的`goroutine`，需要阻塞等待持有这个锁的`goroutine`释放锁。`RWMutex`在某一时刻只能由任意数量的`reader`持有，或者是只被单个的`writer`持有。

RWMtex一共有五个方法：
> Lock/Unlock:

```go
type RWMutex struct {
    w           Mutex  // held if there are pending writers
    writerSem   uint32 // semaphore for writers to wait for completing readers
    readerSem   uint32 // semaphore for readers to wait for completing writers
    readerCount int32  // number of pending readers
    readerWait  int32  // number of departing readers
}

func (rw *RWMutex) Lock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
	// First, resolve competition with other writers.
	rw.w.Lock()
	// Announce to readers there is a pending writer.
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	// Wait for active readers.
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
		race.Acquire(unsafe.Pointer(&rw.writerSem))
	}
}

func (rw *RWMutex) Unlock() {
	if race.Enabled {
		_ = rw.w.state
		race.Release(unsafe.Pointer(&rw.readerSem))
		race.Disable()
	}

	// Announce to readers there is no active writer.
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		race.Enable()
		throw("sync: Unlock of unlocked RWMutex")
	}
	// Unblock blocked readers, if any.
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	// Allow other writers to proceed.
	rw.w.Unlock()
	if race.Enabled {
		race.Enable()
	}
}
```
如果锁已经被 reader 或者 writer 持有，那么，Lock 方法会一直阻塞，直到能获取到锁；Unlock 则是配对的释放锁的方法。

> RLock/RUnlock:

```go
func (rw *RWMutex) RLock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		// A writer is pending, wait for it.
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
	}
}

func (rw *RWMutex) RUnlock() {
	if race.Enabled {
		_ = rw.w.state
		race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
		race.Disable()
	}
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		// Outlined slow-path to allow the fast-path to be inlined
		rw.rUnlockSlow(r)
	}
	if race.Enabled {
		race.Enable()
	}
}
```
如果锁已经被 writer 持有的话，RLock 方法会一直阻塞，直到能获取到锁，否则就直接返回；而 RUnlock 是 reader 释放锁的方法。

> RLocker:

```go
func (rw *RWMutex) RLocker() Locker {
	return (*rlocker)(rw)
}
```
这个方法的作用是为读操作返回一个 Locker 接口的对象。它的 Lock 方法会调用 RWMutex 的 RLock 方法，它的 Unlock 方法会调用 RWMutex 的 RUnlock 方法。

### RWMutex实现原理
`RWMutex`是基于`mutex`实现的。
readers-writers 问题一般有三类，基于对读和写操作的优先级，读写锁的设计和实现也分成三类。
- **Read-preferring**:读优先的设计可以提供很高的并发性，但是，在竞争激烈的情况下可能会导致写饥饿。这是因为，如果有大量的读，这种设计会导致只有所有的读都释放了锁之后，写才可能获取到锁。
- **Write-preferring**: 写优先，如果已经有一个`writer`在等待请求锁的话，它会阻止新来的请求锁的`reader`获取到锁，所以优先保障`writer`。当然，如果有一些reader已经请求到锁，那么新请求的`writer`会等待已经存在的`reader`都释放锁之后才能获取。
- **不指定优先级**:`reader`和`writer`同一个优先级。

RWMutex采用的是**Write-preferring**方案。

```go
type RWMutex struct {
	w   Mutex   //解决多个writer的竞争
	WriterSem   uint32  //writer信号量
	readerSem   uint32  //reader信号量
	readerCount int32   //reader的数量
	readerWait  int32   //writer等待完成的reader的数量
}

const rwmutexMaxReaders = 1 << 30   //定义最大的reader数量
```

> RLock/RUnlock的实现

```go
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
            // rw.readerCount是负值的时候，意味着此时有writer等待请求锁，因为writer优先级高，所以把后来的reader阻塞休眠
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
}
func (rw *RWMutex) RUnlock() {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        rw.rUnlockSlow(r) // 有等待的writer
    }
}
func (rw *RWMutex) rUnlockSlow(r int32) {
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        // 最后一个reader了，writer终于有机会获得锁了
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```
第二行CAS原子操作，对reader数量加1，readerCount有可能是负数，具体含义： 没有`writer`竞争或持有锁时，`readerCount`表示`reader`数量；当有`writer`竞争锁或者持有锁时，`readerCount`不仅表示`reader`数量，还能标识
当前是否有`writer`竞争或者持有锁，当出现这种情况时，请求锁的`reader`会堵塞等待锁的释放（第四行）。
用 32 位的 readerCount 的首位（符号位）来标记是否有 writer 等待。而且设置`rwmutexMaxReaders = 1 << 30`，保留了左边第 2 位，防止 +-1 操作溢出。

调用`RUnlock`的时候，需要将`reader`数量-1（第八行），如果是负数，则表示当前有`writer`竞争锁，在这种情况下，还会调用`rUnlockSlow`方法，检查是否`reader`都
释放了读锁，如果都释放了，就可以唤醒请求写锁的`writer`。

当 writer 请求锁的时候，是无法改变既有的 reader 持有锁的现实的，也不会强制这些 reader 释放锁，它的优先权只是限定后来的 reader 不要和它抢。

> Lock/Unlock

```go

func (rw *RWMutex) Lock() {
    // 首先解决其他writer竞争问题
    rw.w.Lock()
    // 反转readerCount，告诉reader有writer竞争锁
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // 如果当前有reader持有锁，那么需要等待
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
```
RWMutex使用`Mutex`来保证多个`writer`的互斥。

一旦一个`writer`获得了内部的互斥锁（Mutex），就会反转`readerCount`字段，把从原来的正整数修改为负数，让这个字段保持两个含义：既保存了`reader`数量，又表示当前有`writer`。
第5行记录了当前活跃的`reader`数量，如果`readerCount`不是0，就说明当前有持有读锁的`reader`，RWMutex需要把这个当前`readerCount`赋值给`readerWait`字段，同时这个`writer`会进入阻塞状态（第8行）。

每当一个`reader`释放读锁的时候，`readerWait`字段就会 -1，直到所有的活跃的`reader`都释放了读锁，才会唤醒这个`writer`。

> Unlock

```go

func (rw *RWMutex) Lock() {
    // 首先解决其他writer竞争问题
    rw.w.Lock()
    // 反转readerCount，告诉reader有writer竞争锁
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // 如果当前有reader持有锁，那么需要等待
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
```
当一个 `writer`释放锁的时候，它会再次反转`readerCount`字段。可以肯定的是，因为当前锁由`writer`持有，所以，`readerCount`字段是反转过的，并且减去了`rwmutexMaxReaders`这个常数，变成了负数。所以，这里的反转方法就是给它增加 `rwmutexMaxReaders`这个常数值。
在`RWMutex`的`Unlock`返回之前，需要把内部的互斥锁释放。释放完毕后，其他的`writer`才可以继续竞争这把锁。