---
title: kafka 学习笔记
date: 2019-02-03 15:00
category: 后端
tags: [消息队列]
---

kafka 是用于日志处理的分布式消息队列，同时支持离线和在线日志处理。
kafks 在保存消息时根据 topic 进行分类，消息发送者为 producer, 消息接受者为 consumer, 此外 kafka 集群有多个 kafka 实例组成，每个实例称为 broker.无论是 kafka 集群，还是 producer 和 consumer 都依赖于 zookeeper 来保证系统可用性，为集群保存一些 meta 信息。

# 常用术语

记下几个常用的术语

- kafka 维护消息类别的方式是主题(topic)
- 称发布消息到 kafka 主题的进程为生产者(producer)
- 订阅主题，获取消息的进程叫消费者(consumer)
- kafka 是由多个服务器组成的机器，每个服务器称作代理

总的来说，生产者通过网络发布消息到 kafka 集群，kafka 集群将这些消息提供给消费者。

![kafka_service](/image/kafka_service.png)

客户端与服务端通过 TCP 通信，kafka 有 java 客户端，但是许多语言也可以使用

# topic 和 logs

一个 topic 可以认为是一类消息，每个 topic 将分为多个 partition, 每个 partition 在存储层面是 append log 文件。任何发布到此 partition 的消息都会被直接追加到 log 文件的尾部，每条消息在文件中的位置成为 offset(偏移量), offset 为一个 long 型数字，是唯一标记一条信息。kafka 并没有提供其他额外的索引机制来存储 offser, 因为 kafka 几乎不允许对消息进行读和写

一个主题就是消息的类别或者名称，对于每个主题，kafka 集群都管理者一个被分区的日志

# 分区

一个分区就是一个提交日志，每个分区上保留着不断被追加的消息，这些消息时有序的且顺序不可改变。
每个消息都分配了一个序列号 offset, offset 唯一标识了分区上的信息。

在 kafka 中，及时消息被消费，消息也不会被立刻删除。日志文件会根据 broker 中的配置要求，保留一定的时间之后删除，比如 log 文件保留 2 天，那么两天之后，文件会被清除，无论其中的消息是否被消费。kafka 通过这种简单的手段，来释放磁盘空间，以及减少消息消费之后对文件内容改动的 IO 开销。

对于 consumer 而言，需要保存消费信息的 offset, 由 consumer 管理 offset 的保存和使用。当 consumer 正常消费时，offset 将会线性的向前驱动，即消息将依次顺序被消费。事实上 consumer 可以使用任意顺序消费信息，只需要将 offset 重置为任意值。

kafka 集群不需要维护任何 consumer 和 producer 状态信息，这些信息由 zookeeper 保存，因此 producer 和 consumer 的客户端实现非常轻量级，它们可以随时离开，而不会对集群造成额外影响。

partitions 的设计目的有多个。最根本的原因是 kafka 基于文件存储。通过分区，可以将日志内容分散到多个 service 上，避免文件尺寸达到单机磁盘上限，每个 partition 都会被当前 server(kafka 实例) 保存，可以将一个 topic 切分多任意多个 partitions 来保存小心。此外越多的 partitions 意味着可以容纳更多的 consumer, 可有效提升并发消费的能力。

## 分区的目的

对日志进行分区主要有以下几个目的

- 扩容，一个主题可以有多个分区，这使得可以保存比一个机器多得多的数据
- 并行
- 分布式

## 分布式

一个 topic 的多个 partitions 分布在 kafka 集群中的多个 server 上，每个 server 负责 partitions 中消息的读写操作；此外 kafka 还可以配置 partitions 需要备份的个数(replicas), 每个 partition 将会被分到多台机器上，以提高可用性

基于 replicated (冗余) 方案，那么就意味着需要对多个备份进行调度，每个 partition 都有一个机器为 leader, 零个或者多个机器作为 follower, leader 负责所有的读写操作，follow 执行 leader 的指令。如果 leader 失效，那么将会有其他 follower 来接管(成为新的 leader), follower 只是和 leader 跟进，同步消息即可。由此可见作为 leader 的 server 承载了全部的请求压力，因此从集群的整体考虑，有多少个 partitions 就会有多少个 leader, kafka 会将 leader 均衡的分散在每个实例上，来确保整体的性能稳定。

- 发送到 partitions 中的消息将会按照接受的顺序追加到日志
- 对于消费者而言，消费信息的顺序与日志中的消息顺序一致
- 如果 topic 的 replicationfactor (备份因子) 为 N, 那么允许 N - 1 个 kafka 实例失效

# 生产者

producer 将消息发送到指定的 topic 中，同时 producer 也能决定将此消息归属于哪个 partition

# 消费者

传统的消息传递有两种方式

- 队列
  一组消费者从机器上读消息，每个消息只传递给这组消费者中的一个
- 发布 - 订阅
  消息广播到所有消费者，kafka 提供了一个消费组方式

消费者都属于一个消费组，每个消费者中可以有多个消费者。
发送到 topic 中的消息，只会被订阅此 topic 的消费组中的一个消费者消费。如果所有的消费者都有相同的消费组，这种情况近似于 queue 模式。
消息在 consumer 之间负载均衡，如果所有 consumer 都有不同的 group, 那就是纯粹的发布 - 订阅模式，消息会广播给所有的消费者

在 kafka 中，一个 partition 中的消息只会被 group 中的一个 conusmer 消费；每个 group 中 consumer 消息消费相互独立。
可以认为一个 group 是一个订阅者，一个 topic 中的每个 partitions, 只会被一个订阅者中的一个 consumer 消费，不过一个 consumer 可以消费多个 partition 中的消息。kafka 只能保证一个 partition 中的消息被某个 consumer 消费时，消息是有序的。
从 topic 角度来说，消息仍不是有序的。 kafka 的设计原理决定，对于一个 topic, 通一个 group 中不能有多于 partitions 个数的 consumer 同时消费，否则意味着某些 consumer 将无法得到信息。

# 简述

- kafka 将主题下的分区分配给消费组里的消费者，每个分区被一个消费者消费
- 消费者的数量不能超过分区数
- kafka 只能保证分区内的消息是有序的
- 如果需要要消息是全局有序的，可以设置 topic 只有一个 partitions, 但也意味着只能有一个 consumer

producer 发送的消息按照他们发送的顺序追加到主题
消费者看到消息的顺序就是消息在 log 中的存储的顺序

kafka 与传统的消息系统相比，有以下不同

- kafka 被设计为一个分布式系统，易于向外扩展
- 同时为发布和订阅提供高吞吐量
- 支持多订阅者，当失败时能自动平衡消费者
- 将消息持久化到磁盘，因此可用于批量消费，以及实时应用程序
