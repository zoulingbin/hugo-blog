---
title: "Channel源码解析"
date: 2021-05-31T12:38:16+08:00
tags: ["Go", "并发编程"]
author: zoulingbin
---
<!--more-->
### 什么是channel

channel(管道)，go的一种特殊的数据类型，**像是通道（队列），先进先出**，可以通过它们发送类型化的数据在协程之间通信，可以避开所有内存共享导致的坑；通道的通信方式保证了同步性。

### channel 数据结构

[runtime/chan.go](https://github.com/golang/go/blob/master/src/runtime/chan.go#L32)
```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

- qcount: 当前队列/channel中剩余元素个数
- dataqsiz: 环形队列长度，即可以存放的元素个数
- buf: 环形队列指针
- elemsize: 每个元素的大小
- closed: 标识关闭状态
- elemtype: chan中元素类型
- sendx: 队列下标，指示元素写入时存放到队列中的位置
- recvx: 队列下标，指示元素从队列的该位置读出
- recvq: 等待读消息的goroutine队列
- sendq: 等待写消息的goroutine队列
- lock: 互斥锁(标准库sync/mutex)

一个channel只能传递一种类型的值，类型信息存储在hchan数据结构中。
- `elemtype`代表类型，用于数据传递过程中的赋值；
- `elemsize`代表类型大小，用于在`buf`中定位元素位置。
-
[runtime/chan.go](https://github.com/golang/go/blob/master/src/runtime/chan.go#L53)
```go
//双向链表
type waitq struct {
	first *sudog    
	last  *sudog
}
```

[runtime/runtime2.go](https://github.com/golang/go/blob/master/src/runtime/runtime2.go#L344)
```go
type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g    //goroutine
    ...
}
```
从上面的信息可以看出channel是由队列，类型信息、goroutine等待队列组成。


> **环形队列**:

![](https://static.sitestack.cn/projects/GoExpertProgramming/chapter01/images/chan-01-circle_queue.png)

- dataqsiz指示了队列长度为6，即可缓存6个元素；
- buf指向队列的内存，队列中还剩余两个元素；
- qcount表示队列中还有两个元素；
- sendx指示后续写入的数据存储的位置，取值[0, 6)；
- recvx指示从该位置读取数据, 取值[0, 6)；

### channel相关操作

#### 写操作

- 如果等待接收队列recvq不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从`recvq`取出G,并把数据写入，最后把该G唤醒，结束发送过程；
- 如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程；
- 如果缓冲区中没有空余位置，将待发送数据写入G，将当前G加入`sendq`，进入睡眠，等待被读`goroutine`唤醒；

![](https://static.sitestack.cn/projects/GoExpertProgramming/chapter01/images/chan-03-send_data.png)

#### 读操作

- 如果等待发送队列`sendq`不为空，且没有缓冲区，直接从`sendq`中取出G，把G中数据读出，最后把G唤醒，结束读取过程；
- 如果等待发送队列`sendq`不为空，此时说明缓冲区已满，从缓冲区中首部读出数据，把G中数据写入缓冲区尾部，把G唤醒，结束读取过程；
- 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程；
- 将当前`goroutine`加入`recvq`，进入睡眠，等待被写`goroutine`唤醒；

#### 关闭

```go
close(chan)
```
关闭channel时会把recvq中的G全部唤醒，本该写入G的数据位置为nil。把sendq中的G全部唤醒，但这些G会panic。除此之外，panic出现的常见场景还有：
- 关闭值为nil的channel
- 关闭已经被关闭的channel
- 向已经关闭的channel写数据



#### 优雅关闭

*不要从接收端关闭通道，如果通道有多个并发发送方，也不要关闭通道。*

```go
//TODO:示例代码后续补充
```

#### 注意事项

1. 发生 panic 的情况有三种：向一个关闭的 `channel` 进行写操作；关闭一个 `nil` 的 `channel`；重复关闭一个 `channel`。
2. 读、写一个 `nil channel` 都会被阻塞
3. `channel`的泄漏：`groutine`操作`channel`后，一直处于发送或接收阻塞状态，而 `channel` 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 `gouroutine` 会一直处于等待队列中

### reference
《go专家编程》

https://mp.weixin.qq.com/s/ZXYpfLNGyej0df2zXqfnHQ