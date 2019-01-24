---
title: redis 关键技术
date: 2019-01-21 18:00
category: 数据库
tags: [redis]
---

# redis 关键技术

- 事件循环
- 过期 & 逐出
- 持久化
- 主从复制
- pipeline & mget
- redis 集群

# 事件循环

redis 服务在启动后会陷入一个巨大的 while 循环，不停地处理文件事件和时间时间

- 文件事件
  诸如连接上的请求和关闭连接等
- 时间事件
  定时任务

## 文件事件

在多个客户端中实现多路复用(epoll 实现)，接受发来的命令请求，并将命令的执行结果返回客户端

- read 事件
  新建连接，接受请求
- write 事件
  返回结果

逻辑如下

![redis_file_event](/image/redis_file_event.png)

## 时间事件

要在指定时间点运行的事件，多个时间时间以无序链表的形式保存在服务器状态中

![redis_time_event](/image/redis_time_event.png)

常有的时间事件如下

- 更新服务器各类统计信息，如时间，内存占用等
- 数据库后台操作，key 过期清理，数据库 rehash 等
- 关闭，清理失效的客户端连接
- 检查是否需要 RDB dump, AOF 重写
- 主节点，对从节点定期同步
- 集群模式，集群定期同步信息和连接测试

## beforesleep

在进入事件循环前执行

- cluster 集群状态检查
- 处理被 block 卡住的 client, 如一些阻塞请求 BLPOP 等
- 将 AOF buffer 持久化到 AOF 文件

## 流程

beforeSleep -> epollwait -> 处理请求 -> 定时事件 -> ...

在流程中最好时间的步骤为 key hash, 所以减少 key 的大小，最好不要超过 1 MB, 减少耗时命令。

# 逐出

当执行 write 但是内存到达上限时，强制将一些 key 删除

- allkeys 所有 keys
- volatile 设置了过期的 key
- LRU 最近未使用
- random 随机
- ttl 最快过期的

## 逐出特点

- 不是精准算法，而是抽样比对
- 每次写入前判断
- 逐出是阻塞请求的

当 逐出 qps 过高时，会影响正常请求处理

# 过期

当某个 key 到达了 ttl 时间，认为该 key 已经失效

两种方式删除

- 惰性删除
  读写之前判断 ttl, 过期则删除
- 定期删除
  在 redis 定时时间中随机抽取部分 key 判断 ttl

## 过期特点

- 并不一定是按设置时间准时地过期
- 定期删除的时候会判断过期比例，达到阈值才推出

## 建议

建议打散 key 的过期时间，避免大量 key 在同一时间点过期

# 持久化

将内存中的数据 dump 到磁盘文件

## RDB 持久化

- 经过压缩的二进制格式
- fork 子进程 dump 可能会造成瞬间卡顿

## AOF 持久化

- 保存所有修改数据库的命令
- 先写 AOF 缓存，再同步到 AOF 文件
- AOF 重写，达到阈值时触发，减小问价大小

## 应用

应使用 AOF 文件备灾，将数据恢复到最近 3 天的任意小时粒度

# 主从复制

## 主从模式

- 主从节点都可以挂在从节点
- 最终一致性

## 全量同步

- 传递 RDB 文件 & restore 命令重建 kv
- 传递在 RDB dump 过程中的写入数据

## 部分同步

根据 offset 传递积压缓存中的部分数据

![redis_master_slave_copy](/image/redis_master_slave_copy_step.png)

## 主从复制的流程

![master_slave_sync](/image/redis_master_slave_sync_step.png)

# pipline

## client 端

将多个命令缓存起来，缓冲区满了就发送

## redis

处理一个 tcp 连接发来的多个命令，处理完一个发送一个

## twemproxy

既要处理一个 client 连接发来的多个命令，又要捋到同一个下游 redis server 的命令缓存起来一起发送

## 优点

- 节省了往返时间
- 减少了 proxy, redis server 的 IO 次数

![redis_pipline](/image/redis_pipline.png)

# mget

## client

client 使用 mget 命令将多个命令放到一个命令中

## redis

一个命令中处理多个 key, 等所有 key 处理完后组装回复一起发送

# twemproxy

拆分 key 分发到不同 redis server, 需要等待，缓存 mget 中全部回复

## 优点

节省往返时间

## 缺点

- proxy 缓存 mget 结果
- mget 延时是最后一个 key 回复时间，前面的 key 需要等待

## 建议

使用 pipline 代替 mget, 且控制一次请求的命令数量

![redis_mget](/image/redis_mget.png)

# redis 集群

在 redis 中有两种模式建立集群

- 一致性 hash
- redis cluster

## 一致性hash

proxy 作为中间管理节点，分配命令

### 问题

实例宕机，加节点容易造成数据丢失

## redis cluster

节点之间两两通讯维护拓扑信息，有节点数量限制

![redis_cluster](/image/redis_cluster.png)
