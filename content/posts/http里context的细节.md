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
