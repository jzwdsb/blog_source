---
title: kafka exactly 语义
date: 2019-02-25 10:00
category: 后端
tags: [kafka, 消息队列]
---

# kafka 事务

kafka 事务机制的实现主要是为了支持

- `Exactly once` 刚好一次语义
- 操作的原子性
- 有状态操作的原子性

在 kafka 0.11.0.0 之前的版本中只支持 `At least Once` 和 `At Most Once` 语义，尚不支持 `Exactly Once` 语义

## 操作的原子性

操作的原子性是指，多个操作要么全部成功要么全部失败，不存在部分成功部分失败的可能。
实现原子性操作的意义在于

- 操作结果更可控，有助于提升数据一致性
- 便于故障恢复。因为操作时原子的，从故障中恢复时只需要重试该操作(原操作失败) 或者直接跳过该操作(该操作成功), 而不需要记录中间状态，更不需要针对中间状态作特殊处理

# 实现事务机制的几个阶段

## 幂等性发送

producer 有其特有的 `Producer ID`(PID) 和 `Sequence Number`, PID 唯一且透明。
对于每个 PID, 该 producer 发送数据的每个 `<Topic, Partition>` 都对应一个从 0 开始单调递增的 `Sequence Number`.

类似地, Broker 端也会为每个 `<PID, Topic, Partition>` 维护一个序号，并且每次 commit 一条消息时将对应序号递增。对于接受的每条消息，如果其序号比 Broker 维护的序号(即最后一次 commit 时的消息的序号) 大一, 则 Broker 会接受它，否则将其丢弃

- 如果消息序号比 Broker 维护的序号大 1 以上，说明中间有数据尚未写入，即乱序，此时 Broker 拒绝该消息，producer 抛出 `InvalidSequenceNumber`
- 如果消息序号小于等于 Broker 维护的序号，说明该消息已经被保存，即重复消息，Broker 丢弃该消息，Producer 抛出 `DuplicateSequenceNumber`

## 事务性保证

事务保证可以使得应用程序将生产数据和消费数据当做一个原子单元来处理，要么全部成功，要么全部失败，即使该生产或者消费跨多个 `<Topic, Partition>`.
另外有状态的应用也可以保证重启后从断点处继续处理，也即事务恢复。
为了实现这种效果，应用程序必须提供一个稳定的(重启后不变的)唯一的 ID, 也即 `Transaction ID`. `Transaction ID` 与 PID 可能一一对应。区别在于 `Transaction ID` 由用户提供，而 PID 对用户透明。
另外，为了保证新的 Producer 启动后，旧的具有相同 `Transaction ID` 的 Producer 即失效，每次 Producer 通过 Transaction ID 拿到 PID 的同时，还会获取一个单调递增的 epoch. 由于旧的 Producer 的 epoch 比新的 Producer 的 epoch 小，kafka 可以很容易识别出该 Producer 是老的 Producer 并拒绝其请求。

有了 Transaction ID 后，Kafka 可保证

- 跨 Session 的数据幂等发送。当具有相同的 `Transaction ID` 的新的 Producer 实例被创建且工作时，旧的且拥有相同 `Transaction ID` 的 Producer 将不再工作.
- 跨 Session 的事务恢复。如果某个应用实例宕机，新的实例可以保证任何未完成的旧的事务要么 Commit 要么 Abort, 使得新实例从一个正常状态开始恢复

## 完整事务过程

- 找到 `Transaction Coordinator`
- 获取 `PID`
- 开启事务
- Consume-Transform-Produce
- Commit 或者 Abort 事情
