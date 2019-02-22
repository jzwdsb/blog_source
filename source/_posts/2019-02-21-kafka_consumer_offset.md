---
title: kafka consumer 手动设置 offset 的确认问题
date: 2019-02-21 11:00
category: 后端
tags: [kafka, 消息队列]
---

使用 kafka consumer 要手动设置 offset 就需要使用 kafka low level consumer API, 从而更好的控制消费, 比如如下场景

- 同一条消息读多次
- 只读取某个 topic 的部分 partition
- 管理事务，从而确保每条消息被处理一次，且仅被处理一次

与 consumer Group 相比，Low Level Consumer 要求用户做大量的额外工作

- 必须在应用程序中跟踪 offset, 从而确定下一条应该消费哪条消息
- 应用程序需要通过程序获知每个 Partition 的 Leader 是谁
- 必须处理 Leader 的变化

使用 Low Level Consumer 的一般流程如下

- 查找到一个活着的 Broker, 并且找出每个 Partition 的 Leader
- 找出每个 Partition 的 Follower
- 定义好请求，该请求应该能描述应用程序需要哪些数据
- Fetch 数据
- 识别 Leader 的变化，并对其作出必要的响应

# 消费端手动设置 offset 的确认

kafka 如果要自己手动设置 offset, 需要手动 commit 通知 topic 该信息已经消费。

比如接受到了 [1, 2, 3, 4, 5] 消息，这里处理时 [1, 2, 3, 4,] 消息消费失败，没有 commit, 但是消息 5 消费成功，成功 commit, 此时 offset 会重置到 5, 但其实之前的 1, 2, 3, 4 没有丢失，仍然保存在 kafka 中，下一次消费时会直接从 offset 取一个 batchSize 数量的信息，做如上的处理。

## commitSync

该方法是 kafka consumer 手动设置 offset 后，应当调用的 commit 方法，以便 kafka 重置 offset.

官方文档说该方法提交的 offset 会成为 kafka rebalance 和 startup 之后的 first fetch, 也就是说使用该方法提交 offset 之后，offset 会被设置为该值，下次读取会从该 offset fetch. 因此，此时手动提交 offset 需要在消费端自己维护 offset.

# offset 和 consumer position

kafka offset 是一条记录的唯一标识，同样也是 consumer 在这个 partition 的下次应当消费的位置。
关于 consumer 在 partition 实际上有两个概念.
consumer position 保存在一个特殊 topic, `_consumer_offset` 中。

- offset
- committed position

## position

position 为下一条应当交付给 consumer 的记录。将会等于 consumer 已经在这个 partition 中拉取的 offset 的 + 1, 这个值每当 consumer 调用 `poll`(java 接口，其他语言类似) 会自动更新。

## committed position

committed position 是最后一次确认消费的 offset.
默认行为是周期性的自动 commit, 也可以手动调用 consumer 的 `commitSync` 和 `commitAsync` 手动提交 offset.
committed position 是 consumer 宕机恢复时重新读取的 offset, 也是发生 rebalance 时, 原来未消费过该 partitions 的 consumer 分配到该 partitions 时读取的 offset.

# consumer 探活

当 consumer 订阅一个 topic 后，consumer 调用 `poll` 时会自动加入 consumer group.
`poll` API 有保活机制。在底层实现中，会定期发送心跳给 server. 如果在 `session.timeout.ms` 时间间隔后 consumer 仍没有发送心跳，该 consumer 就会被为宕机，触发 rebalance.

consumer 可能会处于 `livelock` 情况，也就是 consumer 仍在定时发送心跳，但是却没有进行消费，就是从其对应的 partitions poll 消息，这种情况下 consumer 处于保活，但是不再活跃，占用资源。同样也有相关机制来防止，如果 consumer 在 `max.poll.interval.ms` 时间间隔后没有调用 `poll` 从 partitions 拉取数据，那么就会被移出其所在 consumer group, 方便其他 consumer 能够消费该 partitions.
当以上情况发生时，该 consumer 会发生一次 commit failure, 由 `commitSync`, 抛出一个 `CommitFailException`