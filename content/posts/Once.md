---
title: "Once的用法与实现"
date: 2021-12-03T17:10:32+08:00
tags: [GO, 并发编程]
description: sync.Once用法记录
author: zoulingbin
---

<!--more-->

### 初始化单例的示例
在平时的开发里，如果初始化单例资源，比如定义包级别的变量，通常会这样写：
```go
package demo

import "time"

var StartTime = time.Now()
```
或者在`init()`函数进行初始化：
```go
package demo

import "time"

var StartTime time.Time

func init() {
	StartTime = time.Now()
}

```
又或者在main函数开始执行的时候进行初始化。这三种方式都是并发安全的。
但很多时候我们要延迟惊醒初始化的时候，会这样写：
```go

package main

import (
    "net"
    "sync"
    "time"
)

// 使用互斥锁保证线程(goroutine)安全
var connMu sync.Mutex
var conn net.Conn

func getConn() net.Conn {
    connMu.Lock()
    defer connMu.Unlock()

    // 返回已创建好的连接
    if conn != nil {
        return conn
    }
    // 创建连接
    conn, _ = net.DialTimeout("tcp", "baidu.com:80", 10*time.Second)
    return conn
}

// 使用连接
func main() {
    conn := getConn()
    if conn == nil {
        panic("conn is nil")
    }
}
```
这种方式实现起来很简单，但是依然有一点性能问题，每次每次请求的时候还是得竞争锁才能读到`conn`，这是比较浪费资源的，因为`conn`如果创建好之后，其实就不需要锁的保护了。

针对这种场景，可以使用`sync.Once`这个并发原语。

### Once的用法
`sync.Once`与`init`的区别：
- init函数是在包首次被加载的时候执行，只执行一次。
- sync.Once是在代码运行中需要的时候执行，只执行一次。
```go
func (o *Once) Do(f func())
```
`sync.Once`只有一个`Do()`方法，`Do()`方法可以被多次调用，但只有第一次调用`Do()`方法时，f参数才会执行。(f必须是无参数无返回值函数)。

因为f参数是一个无参数无返回值的函数，所以可以通过**闭包**的方式引用外部的变量：
```go
var addr = "www.google.com"
var (
	conn net.Conn
    err error
)

once.DO(func() {
	conn, err = net.Dial("tcp", addr)
})
```
在绝大多数的情况下都会使用闭包的方式去初始化一个外部的资源。

### Once的实现
```go
type Once struct {
    done uint32
    m    Mutex
}

func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        // Outlined slow-path to allow inlining of the fast-path.
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```
首先通过原子操作读取`done`字段，如果没有改变则执行doSlow()方法。
`doSlow()`方法里加入了互斥锁，保证只有一个`goroutine`进行初始化，同时利用双检查的机制，再次判断`done`是否为0，如果为0，则是第一次执行，执行完毕后，通过原子操作将`done`设置为1，然后释放锁。
因为使用的是双检查机制，即使此时有多个`goroutine`进入了`doSlow()`方法，后续的`goroutine`会看到`done`的值为·，也不会再次执行f()。这样既保证了并发的`goroutine`会等待f()完成，而且还不会多次执行f()。

### 使用Once可能出现的错误
#### deadlock
在`f()`参数中再次调用当前的这个`Once`的`Do()`的话，会导致死锁。

#### 未初始化
如果`f()`方法执行的时候发生`panic`，或者`f()`执行初始化资源的时候失败了，这个时候，`Once`还是会认为初次执行已经成功了，即使再次调用`Do()`方法，也不会再次执行`f()`。