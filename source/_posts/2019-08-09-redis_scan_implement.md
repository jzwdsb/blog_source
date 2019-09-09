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

$SIZE - 1$ 是一个掩码，用来取 hash(key) 的与 SIZE 的模。例如一个 hash table 的大小为 16, 那么则掩码为 1111(2 进制), key 的 index 为 hash 后的低四位。

# resize

当 hash table resize 时，比如从 16 调整到了 64, 则原来旧的 key 可能在新的 hash table 中的任意位置，新的掩码变为 111111 (2进制)。
假设在此之前我们 使用 1100 的 cursor 进行访问，那么无论前面两位我们填充 0 或者 1 我们都只会遍历到我们在原来的，较小的 hash table 中已经遍历过的 key(因为最后四位是不变的).

对于正在做 resize 的 hash table 来说，会同时存在两个表。此时我们会现在小表上遍历，紧接着在大表上使用相同的 cursor, 使用所有可能的排列组合(比如，目前 cursor 为 101,小表大小为 8, 大表大小为 16, 当在小表上遍历完后, 在再大表上测试 (1)101, 和(0)101 遍历得到的结果)

## 优缺点

这种迭代方法是完全无状态的，并且无需额外的内存开销。

缺点在于

- 一个元素可能会被迭代多次
- 迭代器每次调用都返回多个元素，并要给出遍历的 bucket 中所有 key.
