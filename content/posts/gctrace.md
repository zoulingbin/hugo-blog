---
title: "go trace学习记录"
date: 2021-09-09T16:49:12+08:00
tags: [GO, go tool, go cmd]
description: go trace使用笔记
author: zoulingbin
---

### 什么是go trace
`trace`是go提供的一个工具包，包源码在 `/src/internal/trace`。`trace`可以显示go程序运行中的所有的运行时事件。