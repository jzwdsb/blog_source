---
title: kafka consumer 设计
category: 后端
date: 2019-02-19 16:00
tags: [消息队列, kafka]
mathjax: true
---

# 摘要

主要介绍 kafka high level consumer, consumer group, consumer rebalance, low level consumer 实现的语义，以及适用场景。

# High Level Consumer

对于 kafka 使用者而言，不希望知道消息 offset 等的处理，但是有希望提供一些消息队列都提供的语义，比如同一消息只能被一个 consumer 消费，或者被所有 consumer 消费。因此 kafka High Level Consumer 提供了从 kafka 消费数据的搞高层抽象，从而屏蔽其中细节并提供丰富的语义。

# Consumer Group

High level Consumer 将从某个 partition 读取的最后一条消息的 offset 存于 zookeeper 中。这个 offset 基于客户程序提供给 kafka 的名字来保存，这个名字被称为 Consumer Group. Consumer 是整个 kafka 集群全局的，而非某个 topic 的。每一个 High level Consumer 实例都属于一个 Consumer Group, 若不指定则属于默认的 Group.

传统的 Message queue 都会在消息被消费完后将消息删除，一方面是避免重复消费，另一方面可以保证 queue 的长度比较短，提高效率。而如上文所述，kafka 并不删除已消费的消息，为了实现传统 message queue 纸杯消费一次的语义，kafka 保证每条消息在同一个 Consumer group 里只会被某一个 Consumer 消费。与传统 Message queue 不同的是，kafka 还允许不同的 Consumer group 同时消费同一条信息，这一特性可以为消息的多元化处理提供处理。

# High Level Consumer Rebalance

kafka 同一条信息在一个 consumer group 中只会被一个 consumer 消费，实际上，kafka 保证的是稳定状态下每一个 Consumer 实例只会消费一个或者多个 partition 数据，而某个 partition 的数据只会被某一个特定的 consumer 实例所消费。

kafka 对消息的分配是以 partition 为单位分配的，而非每一条消息作为分配单元。这样设计可能导致一个 consumer group 里面的 consumer 均匀消费数据，优势是每个 consumer 不同都跟大量的 broker 通信，减少通信开销，也降低了分配难度，实现也简单。
由于同一个 partition 里的数据是有序的，这种设计可以保证每个 partition 里面的数据可以被有序消费。

如果每个 consumer group 中的 consumer 数量少于 partition 数量，则至少有一个 consumer 会消费多个 partition 的数据，如果相同，则刚好一个 consumer 对应一个 partition. 如果 consumer 数量多于 partition 数量，会有部分 consumer 无法消费该 topic 中的任何一条消息。

## consumer rebalance 算法

- 将目标 topic 下的所有 partition 排序，存于 $P_G$
- 对于 consumer group 下的所有 consumer 排序，存于 $C_G$, 第 i 个 consumer 记为 $C_i$
- $N = size(P) / size(C)$, 向上取整
- 解除 $C_i$ 对原来分配的 partition 的消费权
- 从 $i \times N$ 到 $(i + 1) \times N - 1$ 个 partition 分配给 $C_i$

目前版本的的 kafka(0.8.2.1) 的 consumer 的 consumer rebalance 的控制策略是由每一个 consumer 通过在 zookeeper 上注册完成 watch 的完成的。每个 consumer 被创建时触发时会触发 consumer group 的 rebalance, 具体启动流程如下

- high level consumer 启动时将其 ID 注册到其 consumer group 下，在 zookeeper 上的路径为 `/consumer/[consumer group]/ids/[consumer id]`
- 在 `/consumers/[consumer group]/ids` 上注册 watch
- 在 `/brokers/ids` 上注册 watch
- 如果 consumer 通过 topic filter 创建信息流，则他会同时在 `/brokers/topics` 上也创建 watch
- 强制自己在其 consumer group 内启动 rebalance 流程。

在这种策略下，每一个 consumer 或者 broker 的增加或者减少都会触发 consumer rebalance. 因为每个 consumer 只负责调整自己所消费的 partition, 为了保证整个 consumer group 的一致性，当一个 consumer 触发了 rebalance 时，该 consumer group 内的其它所有其它 consumer 也应该同时触发 rebalance.

# Low Level Consumer

使用 kafka 的 Low Level Consumer 通常是希望更好的控制数据的消费，比如

- 信息重复消费
- 只读取某个 topic 的部分 patition
- 管理实务，从而确保每条信息被处理一次，且仅被处理一次

与 Consumer group 相比，Low level consumer 要求用户做大量的额外工作。

- 必须在应用程序中跟踪 offset, 从而确定下一条应该消费哪条信息
- 应用程序需要通过程序获知每个 partition 的 leader 是谁
- 必须处理 leader 的变化

使用 low level consumer 的一般流程如下

- 查找到一个活着的 broker, 并且找出每个 partition 的 leader
- 找出每个 partition 的 follower
- 定义好请求，该请求应该能描述应用程序需要哪些数据
- fetch 数据
- 识别 leader 的变化，并对其做出必要的响应