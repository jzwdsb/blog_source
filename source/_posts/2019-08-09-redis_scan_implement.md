---
title: redis scan 命令的实现
category: 数据库
tags: [redis]
description: Redis Scan 命令
date: 2019-08-09
mathjax: true
---


Redis Scan 命令可参照 redis 源码中的 `src/dict.c` 中的 `dictScan` 函数。

# 工作方式

`dictScan` 是用来遍历 redis 字典中所有元素的接口。

使用方式如下，go 语言为例

- 初始化 cusor 为 0, 调用该函数
  
  ```golang
    cursor := 0
    _, cursor, _ = cli.Scan(cursor, "*", 2000)
  ```

- scan 函数完成了一次增量拉取，得到了一个新的 cursor, 使用该 cursor 作为下次 scan 的参数，并重复该过程
- 当 scan 函数返回新的 cursor 为 0 时，表示字典中全部的 key 已经遍历完毕。

redis 的 scan 接z口保证在迭代的时间内，存在于字典中的全部 key 都会被迭代，部分元素可能会被迭代多次。

# 原理

redis scan 的主要思想是从 cursor 的高 bit 位开始递增，通过翻转比特位来实现 cursor 的增长，当 cursor 的所有比特位再次被翻转时，就遍历完毕。
使用这种策略来实现增量拉取是因为 hash 表在迭代的过程中可能会发生 resize.

redis 中的 hash 表总是 2 的整数次幂大小，并且之间用链表连接起来，所以一个 key 的 index 的计算方式为

$$
Pos = hash(key) \\& (SIZE-1)
$$

$SIZE - 1$ 是用来去 hash(key) 的与 SIZE 的模。
