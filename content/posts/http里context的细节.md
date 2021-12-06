---
title: "Http里context的细节"
date: 2021-10-12T17:23:25+08:00
tags: ["Go", "标准库", "网络编程"]
author: zoulingbin
categories: ["网络编程"]
description: net/http源码解析
---

<!--more-->
在那篇[net/http源码解析](https://zoulingbin.github.io/posts/nethttp/)里忽略了关于context的使用细节，这篇文章详细说下这部分。

### context标准库的设计思路
在标准库里，`context`的实现是一个树形结构，在整个树形逻辑链条里，用上下文控制，实现每个节点的信息传递和共享。
在树形逻辑链条里，一个节点拥有两个角色，分别是下游树的管理者，上游树的被管理者。因为需要有对应的两个能力：
- 让整个下游树结束的能力：`CancelFunc()`
- 在上游树结束的时候被通知的能力：`Done()`

### Context是怎么产生的
在`net/http`源码里，第一次出现contex是在`serve.Serve`方法里。

```go
func (srv *Server) Serve(l net.Listener) error {
	...
    ...
	
	baseCtx := context.Background()
	if srv.BaseContext != nil {
		baseCtx = srv.BaseContext(origListener)
		if baseCtx == nil {
			panic("BaseContext returned a nil context")
		}
	}

	var tempDelay time.Duration // how long to sleep on accept failure

	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
		rw, err := l.Accept()
		...
		...
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew, runHooks) // before Serve can return
		go c.serve(connCtx)
	}
}
```

从源码里看到，context是从`baseCtx := context.Background()`产生的。

### context的上下游逻辑
