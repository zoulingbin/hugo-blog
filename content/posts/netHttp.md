---
title: "net/http标准库"
date: 2021-08-03T00:36:22+08:00
tags: ["Go", "标准库", "网络编程"]
author: zoulingbin
categories: ["网络编程"]
description: net/http标准库用法记录
---

### 服务端

示例代码：
```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"strings"
)

func Hello(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()	//解析参数，默认不解析
	fmt.Println(r.Form)
	fmt.Println("scheme", r.URL.Scheme)
	fmt.Println("path", r.URL.Path)
	fmt.Println(r.Form["url_long"])
	for k, v := range r.Form {
		fmt.Println("key:", k)
		fmt.Println("val:", strings.Join(v, ""))
	}
	fmt.Fprintf(w, "hello")		//输出到客户端
}

func main()  {
	http.HandleFunc("/", Hello)
	for {
		log.Fatal(http.ListenAndServe(":9090", nil))
	}
}
```





