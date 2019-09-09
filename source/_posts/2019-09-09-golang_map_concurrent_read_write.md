---
title: golang 中 map 的并发安全
category: golang
date: 2019-09-09 17:00
---

原来以为 golang 这种原生支持并发的语言，内置类型都是并发安全的，结果这次出现了一个 bug, 排查了半天发现是 golang 中的 map 原来是不支持并发的，并发的读和写操作会导致进程 panic.

稍等补充下
