---
title: "pprof学习笔记"
date: 2021-09-09T17:00:06+08:00
tags: [GO, go tool, go cmd]
description: go pprof使用笔记
author: zoulingbin
---

### 什么是pprof
`pprof`是用于可视化和分析性能分析数据的工具，`pprof`以`pprof.proto`读取分析样本的集合，并生成报告以可书画并帮助分析数据。
而`pprof.proto`是一个 `protobuf`的描述文件，它描述了一组`callstack`和`symbolization`信息，作用是统计分析的一组采样的调用栈。
一般来说，性能分析主要关注CPU，内存，磁盘IO，网络这些指标。


`pprof`有一下4种类型：
- CPU profiling（cpu性能分析）：用于分析函数或方法的执行耗时；
- Memory profiling：用于分析程序的内存占用情况；
- Block profiling：用于记录`goroutine`在等待共享资源花费的时间；
- Mutex profiling：用于记录因为锁竞争导致的等待或延迟。

### pprof的采样方式
- runtime/pprof：采集程序（非 Server）的指定区块的运行数据进行分析。
- net/http/pprof：基于HTTP Server运行，并且可以采集运行时数据进行分析。
- go test：通过运行测试用例，并指定所需标识来进行采集。

### pprof的使用

demo代码：
```go
package main

import (
	"log"
	"net/http"
	_ "net/http/pprof"
)


func main() {
	go func() {
		for  {
			log.Println(Add("https://github.com/zoulingbin"))
		}
	}()
	_ = http.ListenAndServe("0.0.0.0:1234", nil)
}

var
	datas[] string

func Add(str string) string {
	data := []byte(str)
	sData := string(data)
	datas = append(datas, sData)
	return sData
}
```
启动服务，访问`http://127.0.0.1:1234/debug/pprof/`，就会看到以下内容：

{{<image src="https://pic.imgdb.cn/item/6139d4c344eaada73902b5b6.png">}}

- allocs：查看过去所有内存分配的样本
- block：查看导致阻塞同步的堆栈跟踪
- cmdline：当前程序的命令行的完整调用路径
- goroutine：查看当前所有运行的`goroutine`堆栈跟踪
- heap：查看活动对象的内存分配情况
- mutex：查看导致互斥锁的竞争持有者的堆栈跟踪
- profile：默认进行30s的`CPU profiling`，得到一个分析用的`profile`文件
- threadcreate：查看创建新os线程的堆栈跟踪
- trace：在后台进行一段那时间的数据采样，返回一个分析用的`trace`文件

`profile`和`trace`得到的`profile`文件，使用命令执行该文件：
```
go tool pprof profile文件地址
go tool trace trace文件地址
```

访问`127.0.0.1:57086/trace`可以看到`go tool trace`生成的图表，执行`go tool pprof`会进入交互模式。

