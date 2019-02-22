---
title: kafka 高可用
category: 后端
date: 2019-02-22 18:00
tags: [kafka, 消息队列]
---

> 本文转自[技术世界](http://www.jasongj.com/), [原文链接](http://www.jasongj.com/2015/04/24/KafkaColumn2)

# 摘要

kafka 在 0.8 以前的版本中，并不提供 High Availablity 机制，一旦一个或多个 Broker 宕机，则宕机期间其上所有 partitions 都无法继续提供服务。若该 Broker 不能恢复，或者磁盘故障，则其数据将丢失。

kafka 从 0.8 版本开始提供的 High Availability 主要包含两方面

- Data Replication
- Leader Election

# Data Repliction

## replica

引入 data repliction 后，一个 partitions 可以多个 replica。
如果这些 replica 在一个 broker 上，那么当这个 broker 宕机时，这个 partitions 仍然不可用，所以这些 replica 会分布在多个 broker 上.
kafka 的 data repliction 需要解决如下问题

- 怎样 propagate 消息
- 在想 producer 发送 ACK 前需要保证有多少个 Replica 已经收到该消息
- 怎样处理某个 Replica 不工作的情况
- 怎样处理 Failed Replica 恢复回来的情况

## propagate 信息

### leader

这些 replica 通过 leader election 选举 leader, 其余为 follower. leader 承载 producer 的 push 请求，并将消息写入本地 log, 其他 follower 从其 leader 中 pull 数据.
leader 会维持一个与其基本保持同步的 Replica 列表，该列表成为 ISR(In-Sync Replica), 每个 Partitions 都会有一个 ISR, 并由 leader 动态维护

### follower

follower 在 pull 到消息并写入 log 后，向 leader 发送 ACK.

### producer commit

在 ISR 列表中的 replica 都已经向 leader 发送 ACK 后，leader 则认为该 message 成功 commit, leader 将增加 HW(high watermark, 该 offset 前 record 认为已经备份) 并向 producer 发送 ACK.
为了提高性能，其他 follower 是在收到消息后立即发送 ACK, 而不是等到写入 log 后，因此已经 commit, leader 只能保证目前存于其他 follower 的内存中，而不是持久化到磁盘中，所以也就不能保证该消息一定能被消费，但是这种场景属于极端场景，比较少见。