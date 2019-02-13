---
title: WAL(Write Ahead Logging) 相关概念
date: 2019-02-12
category: 后端
tags: [数据库]
---

WAL(Write ahead logging) 用于数据库系统中，提供原子性和持久性操作。

在一个使用 WAL 的数据库系统中，所有改动在应用之前都要先写入日志中，同时 redo 和 undo 信息也会保存在日志中。

就是一个简单的概念