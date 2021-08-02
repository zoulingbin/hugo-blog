---
title: "WaitGroup的实现"
date: 2021-07-24T00:36:02+08:00
tags: ["Go", "标准库", "并发编程"]
author: zoulingbin
categories: ["Go并发编程"]
description: 鸟窝大佬的go并发编程阅读记录，WaitGroup的实现原理
---

### WaitGroup的基本用法
WaitGroup提供了三个方法：
```go
func (wg *WaitGroup) Add(delta int)
func (wg *WaitGroup) Done()
func (wg *WaitGroup) Wait()
```
- Add:用来设置WaitGroup的计数值；
- Done:用来将WaitGroup的计数值-1，其实就是调用了`Add(-1)`;
- Wait:调用这个方法的goroutine会一直阻塞，直到WaitGroup的计数值变为0。

