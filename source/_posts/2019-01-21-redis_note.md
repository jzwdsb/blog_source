---
title: redis 初探
category: 数据库
date: 2019-01-21 17:00
mathjax: true
tags: [redis]
---

redis 官网对 redis 的介绍.

> Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker. It supports data structures such as strings, hashes, lists, sets, sorted sets with range queries, bitmaps, hyperloglogs, geospatial indexes with radius queries and streams. Redis has built-in replication, Lua scripting, LRU eviction, transactions and different levels of on-disk persistence, and provides high availability via Redis Sentinel and automatic partitioning with Redis Cluster.

# 前言

Redis 是一个开源的使用 ANSI C 语言编写，支持网络，可基于内存亦可持久化的日志型，key-value 的数据库，并提供多种语言的 API, 源代码见于 [github](https://github.com/antirez/redis)

通常, Redis 将数据存于内存中，或者配置使用虚拟内存。通过两种方式可以实现数据的持久化

- 截图
  将内存中的数据不断写入磁盘
- 日志
  记录每次更新的日志

前者性能较高，但是可能会引起一定程度的数据丢失
后者相反

默认端口 6379

# 特点

支持数据的持久化，可将内存的数据保存到磁盘中，重启的时候方便再次加载使用
支持数据的备份，即 master-slave 模式的数据的备份
性能极高, Redis 读的速度为 110000 t/s, 写的速度为 81000 t/s
所有操作都是原子的

# 数据对象

## String

可以包含任何类型，本质上就是字节序列，比如 jpg 图片或者序列化后的对象，亦可 key 最多可以存储 512 MB

## Hash

一个 String 类型的 field 和 value 的映射表
每个 hash 可以存储 $2^{32} - 1$ 键值对

## list

简单的字符串列表，可以从列表头部(左边) 或者尾部(右边) 添加元素

## set

String 类型的无序组合。通过 hash 实现，所以 添加，删除，查找的复杂度都是 O(1)

## zset

和 set 一样也是 String 类型元素的集合，且不孕允许重复的成员，不同的是每个元素都会关联一个 double 类型的 score.
redis 正是通过 score 来为集合中的成员进行从小到大的排序。添加元素到集合，元素在集合中存在则更新对应的 score

# 操作

关于 redis 的命令可以详见[官网文档](https://redis.io/commands)