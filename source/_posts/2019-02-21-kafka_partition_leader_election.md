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

kafka 的 data repliction 需要解决如下问题

- 怎样 propagate 消息
- 在想 producer 发送 ACK 前需要保证有多少个 Replica 已经收到该消息
- 怎样处理某个 Replica 不工作的情况
- 怎样处理 Failed Replica 恢复回来的情况

## replica

引入 data repliction 后，一个 partitions 可以多个 replica。
如果这些 replica 在一个 broker 上，那么当这个 broker 宕机时，这个 partitions 仍然不可用，所以这些 replica 会分布在多个 broker 上.

## propagate 信息

### leader

这些 replica 通过 leader election 选举 leader, 其余为 follower. leader 承载 producer 的 push 请求，并将消息写入本地 log, 其他 follower 从其 leader 中 pull 数据.
leader 会维持一个与其基本保持同步的 Replica 列表，该列表成为 ISR(In-Sync Replica), 每个 Partitions 都会有一个 ISR, 并由 leader 动态维护

### follower

follower 在 pull 到消息并写入 log 后，向 leader 发送 ACK.

### producer commit

在 ISR 列表中的 replica 都已经向 leader 发送 ACK 后，leader 则认为该 message 成功 commit, leader 将增加 HW(high watermark, 该 offset 前 record 认为已经备份) 并向 producer 发送 ACK.
为了提高性能，其他 follower 是在收到消息后立即发送 ACK, 而不是等到写入 log 后，因此已经 commit, leader 只能保证目前存于其他 follower 的内存中，而不是持久化到磁盘中，所以也就不能保证该消息一定能被消费，但是这种场景属于极端场景，比较少见。
Consumer 读消息也是从 leader 读取，只有被 commit 的消息(offset 低于 HW 的消息), 才会暴露给 Consumer.

## ACK 前需要保证有多少备份

kafka 处理失败需要明确定义一个 Broker 是否活着，对于 Kafka 而言，Kafka 存活包含两个条件，一是它必须维护与 Zookeeper 的 Session(通过 Zookeeper 的 HeartBeat 机制来实现). 二是 Follower 必须能够及时拉取 Leader 的消息，不能落后太多。

Leader 会维护一个 ISR 列表. 如果一个 follower 宕机，或者落后太多，Leader 将把他从 ISR 中移除。这里的落后太多指 Follower 复制的消息落后于 Leader 后的条数超过预定值(`replica.lag.max.messages`, 默认为 4000), 或者 Follower 超过一定时间(`replica.lag.time.max.ms`, 默认为 1000) 未向 Leader 发送 fetch 请求。

kafka 的复制机制既不是完全的同步复制，也不是单纯额异步复制。事实上，同步复制要求所有能工作的 Follower 都要复制完，这条消息才会被认为 commit, 这种复制方式会极大地影响吞吐量。而异步复制方式下，Follower 异步的从 Leader 复制数据，数据只要被 Leader 写入 log 就认为已经 commit, 这种情况下如果 follower 都复制完都落后于 leader, 而如果 Leader 突然宕机，则会丢失数据。而 Kafka 这种使用 ISR 的方式则很好的均衡了数据不丢失以及吞吐量。Follower 可以批量的从 Leader 复制数据，这样极大地提高复制性能(批量写磁盘), 极大减少了 Follower 与 Leader 的差距.

# Leader Election

当 leader 宕机后，如何在 follower 中选举出新的 leader.
一个基本的原则就是，如果 leader 不再了，新的 leader 必须有原来的 leader commit 过的所有消息。

kafka 在 ZooKeeper 中动态维护了一个 ISR(In-sync replics), 这个 ISR 里的所有 Replica 都跟上了 leader, 只有 ISR 里的成员才有被选为 Leader 的可能。在这种模式下，对于 f + 1 个 Replica, 一个 Partitions 能在保证不丢失已经 commit 的消息的前提下容忍 f 个 Replica 的失败。
为了容忍 f 个 Replica 的失败，Majority Vote 和 ISR 在 commit 前需要等待的 Replica 数量是一样的，但是 ISR 需要的总的 Replica 的个数几乎是 Majority Vote 的一半。

## ISR 中所有 Replica 都宕机

当 ISR 中至少有一个 Replica 时，kafka 保证 commit 的数据不丢失，但是如果某个 partition 的所有 Replica 都宕机了，就无法保证数据不丢失了。这种情况下有两种可行方案

- 等待 ISR 中的任意一个 Replica 恢复，并将其选为 Leader
- 选择第一个恢复的 Replica(不一定在 ISR 中) 作为 Leader

这就需要在可用性和一致性当中做出一个简单的折衷。如果一定要等待 ISR 中的 Replica 恢复，那不可用的时间可能会比较长，而且如果这个 Partitions 的 ISR 中的所有 Replica 都无法活过来了，或者数据都丢失了，这个 Partition 将永远不可用。

kafka 选择第二种方式处理这种情况，在未来的版本中，将支持通过配置选择两种方式中的一种。

## 如何选举 leader

所有的 follower 在 zookeeper 上设置一个 watch, 一旦 leader 宕机，其对应的 ephemeral znode 自动删除，此时所有 follower 都尝试创建该节点，创建成功者(zookeeper 保证只有一个能创建成功) 即是新的 Leader, 其它 Replica 即为 Follower.
但是该方法会有 3 个问题

- split-brain
  由 zookeeper 的特性引起，虽然 Zookeeper 能保证 Watch 按顺序触发，但并不能保证同一时刻所有 Replica 看到状态的是一样的，这就可能造成不同的 Replica 的响应不一致
- herd effect
  如果宕机的那个 broker 上的 partition 较多，会造成多个 watch 被触发，造成集群内大量的调整
- Zookeeper
  Zookeeper 负载过重，每个 Replica 都要为此在 Zookeeper 上注册一个 Watch, 当集群规模增加到几千个 Partition 时 Zookeeper 负载会过重

在 kafka 0.8.* 的 leader Election 方案解决上述问题，他在所有 broker 中选出一个 controller, 所有 Partition 的 Leader 都由 Controller 决定。Controller 会将 Leader 的改变直接通过 RPC 的方式(比 Zookeeper queue 更高效) 通知需为此做出相应的 broker, 同时 controller 也负责增删 Topic 以及 Replica 的重新分配。
