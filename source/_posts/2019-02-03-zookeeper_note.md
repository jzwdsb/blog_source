---
title: zookeeper 学习相关
date: 2019-02-03 14:00
category: 后端
tags: [分布式]
---

zookeeper 用于解决分布式系统中数据一致性问题，因为所有的分布式系统都会面临这一问题。
zookeeper 由 yahoo 开发，在开发 hadoop 时将这一问题抽象出来，提出了一个独立和通用的解决方案，就是 zookeeper.

# 设计目标

- 类似于标准文件系统，各个进程之间的协调通过共享分级命名空间，目录或者文件被称为 znodes, 这些数据都会被保存到 zk 的实例内存中
- 以集群形式提供服务
- 有序性
- 高性能
- 极简 API, 一共 7 个(create, delete, exist, get data, set data, get children, sync)

# 结构

zookeeper 服务集群一般由大于等于 3 的单数个服务器构成，在每台服务器内存中维护类似于文件系统的树形数据结构，其中一台作为主控节点，其余作为从属节点。写操作只能由主控节点响应，读可由任务节点响应，如果主控节点宕机，则其余节点可以再选举出新的主控节点
由于主控节点只有一个，所以在写操作上天然存在瓶颈，所以更适合读多写少的应用场景。

## 集群结构

![cluster_structure](/image/zk-arch.png)

## 节点模型

![node_model](/image/zk-node.png)

## 主要应用场景

- 分布式锁
- 服务zhuce
- 配置管理
- 名称服务
- leader 选举