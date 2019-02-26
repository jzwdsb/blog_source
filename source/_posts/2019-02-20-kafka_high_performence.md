---
title: kafka 高性能原理
date: 2019-02-26 14:00
category: 后端
tags: [kafka, 消息队列]
---

# 摘要

从架构层面和具体实现层面分析了 kafka 如何实现高性能

# 架构层面

## partition 实现并行处理

kafka 是基于订阅-发布的消息系统，无论是发布还是订阅，都需要指定 topic, topic 只是一个逻辑上的概念。每个 topic 都包含一个或者或者多个 partition, 不同 partition 可位于不同节点。同时 partition 在物理上对应一个本地文件夹，每个 Partition 包含一个或多个 Segment, 每个 Segment 包含一个数据文件和一个与之对应的索引文件。
Partition 可位于不同机器，因此可以充分利用集群优势，实现机器间的并行处理。另一方面，由于 Partition 在物理上对应一个文件夹，即使多个 Partition 位于同一个节点，也可以通过配置让同一节点上的不同 Partition 置于不同 disk drive 上，从而实现磁盘的并行处理，充分发挥多磁盘的优势。

注: 虽然一个 Partition 可以分为多个 Segment, 但是 Segment 并不能提供并行处理。

## 常用数据复制及一致性方案

### master-slave

- RDBMS 的读写分离即为典型的 Master-Slave 方案
- 同步复制可保证强一致性但会影响可用性
- 异步复制可提供高可用性但是会降低一致性

### WNR

- 主要用于去中心化的分布式系统中。
- N 代表总副本数，W 代表每次写操作要保证的最少写成功的副本数，R 代表每次读至少要读取的副本数
- 当 W + R  > N 时，可保证每次读取的数据至少有一个副本拥有最新的数据
- 多个写皂搓的顺序难以保证，可能导致多副本间的写操作顺序不一致。

### Paxos 及其变种

- Google 的 Chubby, Zookeeper 的原子广播协议

### 基于 ISR 的数据复制方案

kafka 的数据复制是以 Partition 为单位的。而多个备份间的数据复制，通过 Follower 向 Leader 拉取数据完成。类似于 Master-Slave 方案，但是 Kafka 既不是完全的同步复制，也不完全的异步复制，而是基于 ISR 的动态复制方案。

ISR 由 Leader 动态维护。如果 Follower 不能跟上 Leader, 他会被 Leader 从 ISR 中移除，待它又重新跟上 Leader 后，会被 Leader 再次加入 ISR 中。每次改变 ISR 后，Leader 都会将最新的 ISR 持久化到 Zookeeper 中。

0.8.* 版本，如果 Follower 在 `replica.lag.time.max.ms` 时间内未向 Leader 发送 Fetch 请求(也即数据复制请求), 则 Leader 会将其从 ISR 中移除。如果某 Follower 持续向 Leader 发送 Fetch 请求，但是与 Leader 的数据差距在 `replica.lag.max.messages` 以上，也会被 Leader 从 ISR 中移除.

Leader 并不是等到前一条消息被 Commit 才接受后一条消息。Leader 按顺序接受大量消息，最新的一条消息的 offset  被记为 High Watermark. 只有被 ISR 中所有 follower 都复制过去的消息才会 commit, Consumer 只能消费被 Commit 的消息。由于 Follower 的复制时严格按照书序，所以被 commit 的消息之前的消息肯定也已经被 Commit 过。换句话，High Watermark 标记的是 Leader 所保存的最新消息的 offet, commit offset 标记的是最新的可被消费(已同步到 ISR 中的 Follower)消息。而 Leader 对数据的接受与 Follower 对数据的复制是异步进行的，因此会出现 Commit Offset 与 High Watermark 存在一定的情况。

# 具体实现层面

# kafka 在读写上的优化

## 磁盘顺序写

kafka 在将消息写入磁盘时全是顺序写操作，目前大部分磁盘还是机械结构，顺序写要比随机写的效率高很多，避免了大量缓慢的机械运动。
过于频繁的小 I/O 操作会拖慢速度，所以 kafka 会将一批次消息打包到一起批量写回磁盘。

## 充分利用 page cache

通过 MMAP, 利用 page cache, 这是 os 级别的缓存而不是应用级别的，所以 kafka 重启后仍然可用。
读操作可直接在 page cache 内进行。如果消费和生产速度相当，甚至不需要通过物理磁盘(直接通过 page cache) 交换数据.

## 支持多 Disk Drive

Broker 的 `log.dirs` 配置项，允许配置多个文件夹。如果机器上有多个 Disk Drive, 可将不同的 Disk 挂载到不同的目录，然后将这些目录都配置到 `log.dirs` 里，kafka 会尽可能将不同的 Partition 分配到不同的目录，也即不同的 Disk 上面，充分利用多 Disk 的优势.

## 零拷贝

当需要网络传输日志时，比如传输持久性日志块，常规的接口从传输需要四次拷贝。
在使用 MMAP 时，可以使用 Linux 提供的传输接口 `sendfile` 系统调用，直接从页缓存复制到 socket 中进行网络传输。
传统的套接字发送接口中间需要发生四次数据拷贝。

## 减少网络开销

通过以下几种手段减少网络开销

- 批处理
- 数据压缩降低网络负载
- 高效的序列化方式
