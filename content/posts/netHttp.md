---
title: "net/http标准库"
date: 2021-08-03T00:36:22+08:00
tags: ["Go", "标准库", "网络编程"]
author: zoulingbin
categories: ["网络编程"]
description: net/http源码解析
---

<!--more-->

使用net/http库来创建一个http服务：
```go
package main

import (
	"fmt"
	"html"
	"net/http"
)

type S struct {}

func (*S) ServeHTTP(w http.ResponseWriter, r *http.Request){

}

func main() {
	http.Handle("/foo", &S{})
	http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "hello, %q",html.EscapeString(r.URL.Path))
	})
	http.ListenAndServe(":8000", nil)
}
```

通过`go doc net/http | grep "^func"`命令查询出`net/http`库所有的对外函数：
```go
func CanonicalHeaderKey(s string) string
func DetectContentType(data []byte) string
func Error(w ResponseWriter, error string, code int)
func Get(url string) (resp *Response, err error)
func Handle(pattern string, handler Handler)
func HandleFunc(pattern string, handler func(ResponseWriter, *Request))
func Head(url string) (resp *Response, err error)
func ListenAndServe(addr string, handler Handler) error
func ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error
func MaxBytesReader(w ResponseWriter, r io.ReadCloser, n int64) io.ReadCloser
func NewRequest(method, url string, body io.Reader) (*Request, error)
func NewRequestWithContext(ctx context.Context, method, url string, body io.Reader) (*Request, error)
func NotFound(w ResponseWriter, r *Request)
func ParseHTTPVersion(vers string) (major, minor int, ok bool)
func ParseTime(text string) (t time.Time, err error)
func Post(url, contentType string, body io.Reader) (resp *Response, err error)
func PostForm(url string, data url.Values) (resp *Response, err error)
func ProxyFromEnvironment(req *Request) (*url.URL, error)
func ProxyURL(fixedURL *url.URL) func(*Request) (*url.URL, error)
func ReadRequest(b *bufio.Reader) (*Request, error)
func ReadResponse(r *bufio.Reader, req *Request) (*Response, error)
func Redirect(w ResponseWriter, r *Request, url string, code int)
func Serve(l net.Listener, handler Handler) error
func ServeContent(w ResponseWriter, req *Request, name string, modtime time.Time, ...)
func ServeFile(w ResponseWriter, r *Request, name string)
func ServeTLS(l net.Listener, handler Handler, certFile, keyFile string) error
func SetCookie(w ResponseWriter, cookie *Cookie)
func StatusText(code int) string
```

去掉New和Set开头的函数，剩下的函数可以分为这三大类：
- 为服务端提供创建HTTP服务的函数，名字中一般包含`Serve`；
- 为客户端提供调用HTTP服务的类库，以http的`method`同名，比如Get，Post，Head等；
- 提供代理的一些函数，比如`ProxyURL`,`ProxyFromEnvironment`等。

先分析`Serve`函数：
```go
func Serve(l net.Listener, handler Handler) error
func ServeContent(w ResponseWriter, req *Request, name string, modtime time.Time, ...)
func ServeFile(w ResponseWriter, r *Request, name string)
func ServeTLS(l net.Listener, handler Handler, certFile, keyFile string) error
func ListenAndServe(addr string, handler Handler) error
func ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error
```

先从启动服务的`http.ListenAndServe`函数开始：
```go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

通过示例代码和`http.ListenAndServe`源码可以看到，该方法创建一个`Server`结构，调用`Server.ListenAndServe`对外提供服务。接下来再看`Server.ListenAdnServer`源码：
```go
func (srv *Server) ListenAndServe() error {
	if srv.shuttingDown() {
		return ErrServerClosed
	}
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(ln)
}
```
`Server.ListenAdnServer`方法先定义了监听信息，再调用`Serve`方法。在`Serve`方法中，用一个for循环，通过`l.Accept`不断接收从客户端传来的请求连接。当接收到一个新的请求时，通过`srv.Conn`
创建一个连接结构(http.conn)，并创建一个`goroutine`去处理这个请求。

`````go
func (srv *Server) Serve(l net.Listener) error {
	if fn := testHookServerServe; fn != nil {
		fn(srv, l) // call hook with unwrapped listener
	}

	origListener := l
	l = &onceCloseListener{Listener: l}
	defer l.Close()

	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}

	if !srv.trackListener(&l, true) {
		return ErrServerClosed
	}
	defer srv.trackListener(&l, false)

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
		if err != nil {
		    ...
		    ...
		}
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
`````

从`c.serve(connCtx)`开始就是单个连接的逻辑了。`c.serve`函数先判断本次请求时候需要升级为https，然后创建都文本的`reader`和写文本的`buffer`，再进一步读取本次请求数据，然后调用`serverHandler{c.server}.ServeHTTP(w, w.req)`来处理这次请求。

```go
type ServeMux struct {
    mu    sync.RWMutex
    m     map[string]muxEntry
    es    []muxEntry // slice of entries sorted from longest to shortest.
    hosts bool       // whether any patterns contain hostnames
}


type serverHandler struct {
	srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```

`DefaultServeMux`是一个`ServeMux`结构的实例，是一个多路复用器。在最前面调用`http.ListenAndServe()`时，如果没有传入第二个参数hanler，则`handler = DefaultServeMux`。
启动http服务的时候，可以使用`http.HandleFunc()`来指定路由和处理函数。

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}

func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    if handler == nil {
        panic("http: nil handler")
    }
    mux.Handle(pattern, HandlerFunc(handler))
}

func (mux *ServeMux) Handle(pattern string, handler Handler) {
    mux.mu.Lock()
    defer mux.mu.Unlock()

    if pattern == "" {
        panic("http: invalid pattern")
    }
    if handler == nil {
        panic("http: nil handler")
    }
    if _, exist := mux.m[pattern]; exist {
        panic("http: multiple registrations for " + pattern)
    }

    if mux.m == nil {
        mux.m = make(map[string]muxEntry)
    }
    e := muxEntry{h: handler, pattern: pattern}
    mux.m[pattern] = e
    if pattern[len(pattern)-1] == '/' {
        mux.es = appendSorted(mux.es, e)
    }

    if pattern[0] != '/' {
        mux.hosts = true
    }
}
```
通过源码可以看到，最终的调用是`ServeMux.Handle`方法。`ServeMux.Handle`是一个非常简单的`map`实现，`key`是路径(pattern)，`value`是这个`pattern`对应的处理函数。它是通过`mux.match(path)`寻找对应的`Handler`，
也就是`DefaultServeMux`内部的map中直接根据`key`寻找到`value`的。

### 总结
抛去关于HTTPS的处理，具体的刘城逻辑就是：
- 创建http服务是通过创建一个`Server`结构完成的；
- `Server`在for循环里不断监听每一个连接；
- 每个连接默认开启一个`goroutine`去处理；
- `Server`如果没有设置处理函数`Handler`，默认使用`DefaultServeMux`处理请求；
- `DefaultServeMux`是使用map来存储和查找路由规则。

{{<image src = "https://s3.bmp.ovh/imgs/2021/10/58a48ae3bce26124.png">}}