---
title: redis scan 命令的实现
category: 数据库
tags: [redis]
description: Redis Scan 命令
date: 2019-08-09 
---

Redis Scan 命令可参照 redis 源码中的 `src/dict.c` 中的 `dictScan` 函数。

# 工作方式

`dictScan` 用来遍历
