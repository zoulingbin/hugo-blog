---
title: "GMP调度"
date: 2021-06-29T17:56:43+08:00
tags: [GO, 并发编程]
description: gmp调度分析
author: zoulingbin
---

<!--more-->
### 什么是gmp模型
Golang 语言的调度器是基于协程调度模型 GMP，即 goroutine（协程）、processor（处理器）、thread（线程），通过三者的相互协作，实现在用户空间管理和调度并发任务。
{{< image src="/posts/640.png" caption="GMP模型" height="600px" width="600px">}}

> G

G是`goroutine`的缩写,每个goroutine都有自己的栈空间，定时器，初始化的栈空间在2k左右，空间会随着需求增长。G的数量没限制，但受内存影响。

> M

抽象化代表内核线程，记录内核线程栈信息，当goroutine调度到线程时，使用该goroutine自己的栈信息。Go 语言中，M 的默认数量限制是 10000，如果超出则会报错

> P

Processor，处理器，一般`P`的数量就是处理器的核数，可以通过`GOMAXPROCS`进行修改。

#### G-M-P三者关系
G-M-P三者的关系与特点：
- 每一个`P`保存着一个协程G的队列，即`p`的本地队列。
- 除了每个`P`自身保存的`G`的队列外，调度器还拥有一个全局的`G`队列
- `M`的数量和P不一定匹配，可以设置很多`M`，`M`和`P`绑定后才可运行，多余的M处于休眠状态。 
- 无论在哪个`M`中创建了一个`G`，只要`P`有空闲的，就会引起新`M`的创建

三者关系：G需要绑定在M上才能运行，M需要绑定P才能运行。
  
### 线程调度原理
- N:1模型：多个用户空间线程在1个内核空间线程上运行。优势是上下文切换非常快，因为这些线程都在内核态运行，但是无法利用多核系统的优点。
- 1:1模型：1个内核空间线程运行一个用户空间线程。这种充分利用了多核系统的优势但是上下文切换非常慢，因为每一次调度都会在用户态和内核态之间切换。POSIX线程模型(pthread)就是这么做的。
- M:N模型：内核空间开启多个内核线程，一个内核空间线程对应多个用户空间线程。效率非常高，但是管理复杂。

### GMP调度原理
在旧版本里，go的goroutine调度模型是GM。

{{< image src="https://cdn.learnku.com/uploads/images/202003/11/58489/uWk9pzdREk.png" caption="GM模型" height="600px" width="600px">}}

在老调度器的缺点主要是，多个线程（M）从全局队列里获取goroutine时，需要频繁的加锁，形成了激烈的锁竞争。

{{< image src="https://cdn.learnku.com/uploads/images/202003/11/58489/Ugu3C2WSpM.jpeg" caption="GMP模型" height="600px" width="600px">}}

全局队列里存放着等待运行的G。每个`P`都维护自己的一个本地队列，这样就可以解决每次执行`G`都要去全局队列取`G`，导致的频繁抢占锁的问题。每个`P`都有一个`runnext`。`runnext`实际上只能指向一个`G`,可以理解为特殊的队列。如果 runnext 为空，那么 goroutine 就会顺利地放入 runnext，接下来，它会以最高优先级得到运行，即优先被消费。

如果`runnext`不为空，那就先负责把`runnext`上的`old G`踢走，再把`new G`放上来。被踢走的`G`会被放到本地队列，本地队列的容量是256，如果本地队列满了，则会将本地队列的一半数量的`G`放到全局队列里，被其他的`M`消费掉。当本地队列为空时，M会执行全局队列的`G`。
当全局队列为空时，空闲的M会执行其他P的本地队列里的`G`。

`M`执行过程中，随时会发生上下文切换。当发生上线文切换时，需要对执行现场进行保护，以便下次被调度执行时进行现场恢复。Go调度器`M`的栈保存在`G`对象上，只需要将`M`所需要的寄存器(SP、PC等)保存到`G`对象上就可以实现现场保护。当这些寄存器数据被保护起来，就随时可以做上下文切换了，在中断之前把现场保存起来。如果此时`G`任务还没有执行完，`M`可以将任务重新丢到`P`的队列，等待下一次被调度执行。当再次被调度执行时，`M`通过访问`G`的SP、PC寄存器进行现场恢复，从上次中断位置继续执行。


一个`G`最多占用CPU 10ms，防止其他`G`被饿死。

#### 调度器的生命周期
{{< image src="/posts/gmp.png" caption="gmp生命周期" height="600px" width="600px">}}

> M0

 M0 是启动程序后的编号为 0 的主线程，这个 M 对应的实例会在全局变量 runtime.m0 中，不需要在 heap 上分配，M0 负责执行初始化操作和启动第一个 G， 在之后 M0 就和其他的 M 一样了。

> G0

`G0`是每次启动一个`M`都会第一个创建的`goroutine`，`G0`仅用于负责调度的`G`，`G0`不指向任何可执行的函数，每个`M`都会有一个自己的`G0`。在调度或系统调用时会使用`G0`的栈空间，全局变量的`G0`是`M0`的`G0`。



### reference
https://mp.weixin.qq.com/s/DZVn-5n-yB-swE0J4CjcbQ

https://mp.weixin.qq.com/s/an7dml9NLOhqOZjEGLdEEw

https://mp.weixin.qq.com/s/5E5V56wazp5gs9lrLvtopA

https://learnku.com/articles/41728 (图片来源)