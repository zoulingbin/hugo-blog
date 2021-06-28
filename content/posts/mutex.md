---
title: "Mutex解析"
date: 2021-06-28T15:27:17+08:00
tags: ["Go", "标准库", "并发编程"]
author: zoulingbin
description: 鸟窝大佬的go并发编程阅读记录，从初版mutex到第四版的源码分析
categories: ["Go并发编程"]
---

<!--more-->

### mutex架构演进的四个阶段：
![](https://note.youdao.com/yws/api/personal/file/WEB96b1758fed38c543ffda0ee3feefabb8?method=download&shareKey=0dbc3b4104ff2d8cf2a821243261fc5e)

### 初版

“初版”的`Mutex`使用一个`flag`来表示锁是否被持有，实现比较简单；后来照顾到新来的`goroutine`，所以会让新的 goroutine 也尽可能地先获取到锁；那么，接下来就是第三阶段“多给些机会”，照顾新来的和被唤醒的 `goroutine`；但是这样会带来饥饿问题，所以目前又加入了饥饿的解决方案，也就是第四阶段“解决饥饿”。

**初版的互斥锁**
```go
   // CAS操作，当时还没有抽象出atomic包
    func cas(val *int32, old, new int32) bool
    func semacquire(*int32)
    func semrelease(*int32)
    // 互斥锁的结构，包含两个字段
    type Mutex struct {
        key  int32 // 锁是否被持有的标识
        sema int32 // 信号量专用，用以阻塞/唤醒goroutine
    }
    
    // 保证成功在val上增加delta的值
    func xadd(val *int32, delta int32) (new int32) {
        for {
            v := *val
            if cas(val, v, v+delta) {
                return v + delta
            }
        }
        panic("unreached")
    }
    
    // 请求锁
    func (m *Mutex) Lock() {
        if xadd(&m.key, 1) == 1 { //标识加1，如果等于1，成功获取到锁
            return
        }
        semacquire(&m.sema) // 否则阻塞等待
    }
    
    func (m *Mutex) Unlock() {
        if xadd(&m.key, -1) == 0 { // 将标识减去1，如果等于0，则没有其它等待者
            return
        }
        semrelease(&m.sema) // 唤醒其它阻塞的goroutine
    }    
```

CAS 指令将**给定的值和一个内存地址中的值进行比较**，如果它们是同一个值，就使用新值替换内存地址中的值，这个操作是原子性的。**原子性保证这个指令总是基于最新的值进行计算，如果同时有其它线程已经修改了这个值，那么，CAS 会返回失败**。


`mutex`结构体包含两个字段：
- **key**:是一个 `flag`，用来标识这个排外锁是否被某个`goroutine` 所持有，如果`key`大于等于`1`，说明这个排外锁已经被持有；
- **sama**:是个信号量变量，用来控制等待`goroutine`的阻塞休眠和唤醒。
  ![](https://note.youdao.com/yws/api/personal/file/WEBb21b2a0294b7db6078612944963e771a?method=download&shareKey=b03cb5a7dc5a11c4e73a35d1e4facd68)

调用`Lock`请求锁时，通过`xadd`方法进行**CAS操作**，xadd方法循环执行CAS操作直到成功，保证对`key`加1的操作成功完成。如果比较幸运，锁没有被别的 goroutine 持有，那么，`Lock`方法成功地将`key`设置为 1，这个`goroutine` 就持有了这个锁；如果锁已经被别的`goroutine`持有了，那么，当前的 goroutine 会把 key 加 1，而且还会调用`semacquire` 方法，使用信号量将自己休眠，等锁释放的时候，信号量会将它唤醒。

`CAS`:compare-and-swap,atomic包里的`CompareAndSwapInt32()`方法，比较并交换。

持有锁的`goroutine`调用`Unlock`释放锁时，它会将`key`减 1。如果当前没有其它等待这个锁的 goroutine，这个方法就返回了。但是，如果还有等待此锁的其它 goroutine，那么，它会调用`semrelease`方法，利用信号量唤醒等待锁的其它 goroutine 中的一个。

初版的MUtex利用**CAS原子操作**，对`key`标志量进行设置。`key`不仅仅标识了锁是否被`goroutine`所持有，还记录了当前持有和等待获取锁的`goroutine`的数量。

**`Unlock`方法可以被任意的`goroutine`调用释放锁，即使是没持有这个互斥锁的`goroutine`，也可以进行这个操作**。这是因为，Mutex 本身并没有包含持有这把锁的 goroutine 的信息，所以，Unlock 也不会对此进行检查。

所以我们在使用mutext的时候，必须保证goroutine尽可能不去释放自己未持有的锁，遵循**谁申请，谁释放**的原则。

### 第二版
```go
type Mutex struct{
    state int32
    sema uint32
}

const (
    mutextLocker = 1 << iota    //持有锁的标记
    mutextWoken         //唤醒标记
    mutexWaiterShift = iota //阻塞等待的waiter数量
)
```

Mutex还是两个字段，但是第一个字段已经改成`state`，是一个复合字段，表示多个意义。这个字段的第一位标识这个锁是否被持有，第二位标识是否又唤醒的`goroutine`，剩余的位数代表等待此锁的`goroutine`数。
请求锁的goroutine有两种：
- 新goroutine
- 被唤醒的goroutine

请求锁的goroutine有两类：

请求锁的goroutine | 当前锁被持有 | 当前锁未被持有
---|--- |---|
新来的goroutine| waiter++；休眠 |获取到锁
被唤醒的goroutine | 清除mutexWoken标志；重新休眠，加入等待队列|清除mutexWoken标志，获取到锁


**请求锁的方法 Lock实现：**
```go
func (m *Mutex) Lock() { 
    // Fast path: 能够直接获取到锁 
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return 
    } 
    awoke := false 
    for { 
        old := m.state 
        new := old | mutexLocked // 新状态加锁 
        if old&mutexLocked != 0 {
            new = old + 1<<mutexWaiterShift
        }
        if awoke { 
            // goroutine是被唤醒的， 
            // 新状态清除唤醒标志 
            new &^= mutexWoken 
        } 
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
        //设置新状态 
            if old&mutexLocked == 0 { 
                // 锁原状态未加锁 
                break 
                    
            } 
            runtime.Semacquire(&m.sema) // 请求信号量 
            awoke = true
        } 
        
    }
}
```
先是通过CAS检测`state`字段中的标志，如果没有goroutine持有锁，也没有等待持有锁的goroutine，那么当前的goroutine可以直接获得锁。

如果`state`不是零值，那么通过一个循环检查。for循环是不断尝试获取锁，如果获取不到，就通过`runtime.Semacquire(&m.sema)`休眠，休眠醒来之后`awoke`设置为`true`，尝试争抢锁。

第九行` new := old | mutexLocked `将当前的flag设置为加锁状态，如果能成功通过CAS把这个新值赋到`state`，就代表抢占锁成功了。

**释放锁`Unlock()`方法**：
```go
   func (m *Mutex) Unlock() {
        // Fast path: drop lock bit.
        new := atomic.AddInt32(&m.state, -mutexLocked) //去掉锁标志
        if (new+mutexLocked)&mutexLocked == 0 { //本来就没有加锁
            panic("sync: unlock of unlocked mutex")
        }
    
        old := new
        for {
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 { // 没有等待者，或者有唤醒的waiter，或者锁原来已加锁
                return
            }
            new = (old - 1<<mutexWaiterShift) | mutexWoken // 新状态，准备唤醒goroutine，并设置唤醒标志
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime.Semrelease(&m.sema)
                return
            }
            old = m.state
        }
    }
```

