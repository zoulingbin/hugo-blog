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
`Server.ListenAdnServer`方法先定义了监听信息，再调用`Serve`方法。在`Serve`方法中，用一个for循环，通过`l.Accept`不断接收从客户端传来的请求连接。当接收到一个新的请求时，通过`srv.Conn``
创建一个连接结构(http.conn)，并创建一个`goroutine`去处理这个请求。







