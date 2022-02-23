---
title: "Raft算法原理"
date: 2022-02-23T11:58:15+08:00
tags: ["分布式", "算法"]
author: zoulingbin
description: "记录raft原理的个人理解"
---
<!--more-->
## 算法的基本流程
### Raft算法概述

> raft算法在维基百科上是这样解释的：raft是一种用于替代Paxos的共识算法。 相比于Paxos，Raft的目標是提供更清晰的逻辑分工使得算法本身能被更好地理解，同时它安全性更高，并能提供一些额外的特性。 Raft能为在计算机集群之间部署有限状态机提供一种通用方法，并确保集群内的任意节点在某种状态转换上保持一致。

`raft`是实现分布式共识的一种算法，主要用来管理日志复制的一致性。不同与`Paxos`直接从分布式一致性问题出发推导出来，`raft`则是从多副本状态机的角度提出，用于管理多副本状态机的日志复制。
Raft算法由leader节点来处理一致性问题。

`Raft`将共识问题分成如下几个问题：
- **Leader election** leader选举：有且仅有一个leader节点，如果leader宕机，通过选举机制选出新的leader；
- **Log replication** 日志复制：leader从客户端接收数据更新/删除请求，然后日志复制到follower节点，从而保证集群数据的一致性；
- **Safety** 安全性：如果某个节点已经将一条提交过的数据输入raft状态机执行了，那么其它节点不可能再将相同索引的另一条日志数据输入到raft状态机中执行。

`raft`算法需要一直保持这几个属性：
- **Election Safety** 选举安全性：在一个任期内只能存在最多一个leader节点；
- **Leader Append-Only** Leader节点上的日志为只添加：leader节点永远不会删除或者覆盖本节点上面的日志数据，leader节点上写日志的操作只可能是添加操作；
- **Log Matching** 日志匹配性：如果两个节点上的日志，在日志的某个索引上的日志数据其对应的任期号相同，那么在两个节点在这条日志之前的日志数据完全匹配。
- **Leader Completeness** leader完备：如果一条日志在某个任期被提交，那么这条日志数据在leader节点上更高任期号的日志数据中都存在。
- **State Machine Safety** 状态机安全性：如果某个节点已经将一条提交过的数据输入raft状态机执行了，那么其它节点不可能再将相同索引的另一条日志数据输入到raft状态机中执行。

## Reference
https://shuwoom.com/?p=826

http://dockone.io/article/2434665

https://www.codedump.info/post/20180921-raft/

https://zhuanlan.zhihu.com/p/32052223