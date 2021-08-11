---
title: "Kong使用笔记"
date: 2021-08-12T00:07:11+08:00
description: kong使用笔记
author: zoulingbin
tags: [gateway]
---

### 什么是kong
Kong是云原生、高效、可扩展、分布式的微服务抽象层，被称为API网关，或者API中间件。kong通过插件的形式提供负载均衡，身份验证，日志记录，速率限制等功能。

![kong.png](https://i.loli.net/2021/08/12/iHvf5Je16bhxmWE.png)

### kong术语

#### Upstream

负载均衡策略，类似nginx的upstream模块。upstream就是一个虚拟的服务，可以用于配置多个`target`用来实现负载均衡。当部署集群时，一个单独的地址不足以满足的时候，我们可以使用Kong的
`upstream`来进行设置。
```
# 添加upstream
curl -i -X POST --url http://localhost:8001/upstreams/ --data 'name=test-upstream
```

#### Target
目标IP地址/主机名，最终处理请求的backend服务，端口表示后端服务的实例。每个可以有`upstream`多个target。由于`upstream`维护`target`的更改历史记录，
所以无法删除或者修改`target`，如果要金庸目标，需要发布一个新的target`weight = 0`，或者使用DELETE完成相同的操作。

#### Services
服务实体是上游服务的抽象，服务的主要属性是它的 `URL`（其中，Kong 应该代理流量），其可以被设置为单个串或通过指定其`protocol`，`host`，`port` 和`path`。路由是 Kong 的入口点，并定义匹配客户端请求的规则。一旦匹配路由，Kong 就会将请求代理到其关联的服务。
新增service后，kong会自动分配一个id值，该id作为service的唯一标识，可用于后期修改，查看，绑定`route`，`upstream`。
```
#通过 Admin API 添加 Service
curl -i -X POST --url http://localhost:8001/services/ --data 'name=example-service' --data 'url=http://mockbin.org'
```

#### routes
路由实体定义规则以匹配客户端的请求。每个`Route`与一个`Service`相关联，一个服务可能有多个相关联的路由，与给定路由匹配的每个请求都将代理到其关联的`Service`上。
`Route`可配置的字段有：`paths`,`methods`, `hosts`。
`Route`和`Service`的组合提供了一种强大的路由机制，可以在kong中定义细粒度的入口点，从而使基础架构路由到不同的上游服务。

#### Consumer
Consumer对象表示服务的使用者或用户。

#### Plugin
插件，可以是全局的，绑定到`service`，`route`，`consumer`。插件实体表示将在 HTTP请求/响应生命周期 期间执行的插件配置。它是为在 Kong 后面运行的服务添加功能的，例如身份验证或速率限制。
将插件配置添加到服务时，客户端向该服务发出的每个请求都将运行所述插件。如果某个特定消费者需要将插件调整为不同的值，你可以通过创建一个单独的插件实例，通过 service 和 consumer 字段指定服务和消费者。

#### 对应关系

- Upstream:target -> 1:n
- Service:Upstream -> 1:1 or 1:0 (直接指向具体的`Target`，相当于不做负载均衡)
- Service:Route -> 1:n

### Konga
`konga`是一个开源的[部署管理面板](https://github.com/pantsel/konga)。