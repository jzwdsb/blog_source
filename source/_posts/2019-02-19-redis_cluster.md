---
title: redis cluster 节点
category: 数据库
date: 2019-02-19 19:00
tags: [redis]
---

# redis 节点相关

每个 redis cluster 中的节点都有集群当前的配置信息，给出以下信息

- 当前已知的节点
- 与其他节点的连接的信息
- 标志
- 属性
- 赋予的 slots

`cluster nodes` 命令可以得到与当前终端连接的节点所属的集群上面的信息。格式与 redis 在硬盘上保存信息的格式相同。

# 序列化格式

## 示例

```plain
07c37dfeb235213a872192d90877d0cd55635b91 127.0.0.1:30004 slave e7d1eecce10fd6bb5eb35b9f99a514335d9ba9ca 0 1426238317239 4 connected
67ed2db8d677e59ec4a4cefb06858cf2a1a89fa1 127.0.0.1:30002 master - 0 1426238316232 2 connected 5461-10922
292f8b365bb7edb5e285caf0b7e6ddc7265d2f4f 127.0.0.1:30003 master - 0 1426238318243 3 connected 10923-16383
6ec23923021cf3ffec47632106199cb7f496ce01 127.0.0.1:30005 slave 67ed2db8d677e59ec4a4cefb06858cf2a1a89fa1 0 1426238316232 5 connected
824fe116063bc5fcf9f4ffd895bc17aee7731ac3 127.0.0.1:30006 slave 292f8b365bb7edb5e285caf0b7e6ddc7265d2f4f 0 1426238317741 6 connected
e7d1eecce10fd6bb5eb35b9f99a514335d9ba9ca 127.0.0.1:30001 myself,master - 0 0 1 connected 0-5460
```

## 格式

输出格式为如下的 CSV 格式

```plain
<id> <ip:port> <flags> <master> <ping-sent> <pong-recv> <config-epoch> <link-state> <slot> <slot> ... <slot>
```

- id
  长度为 40 的随机字符串，在一个创建一个节点生成并不再改变
- ip:port
  该节点的 ip 和 port
- flags
  由 `,` 分割的 `myself`, `master`, `slave`, `fail?`, `fail`, `handshake`, `noaddr`, `noflags`
- master 如果该节点非 master 节点且 master 节点已知，则这里应为 master id, 否则为 -
- ping-sent unix 当前发送活跃 ping 的毫秒时间戳
- pong-recv 最后一次收到 pong 的时间戳
- config-epoch 节点的目前的配置版本
- link-state 集群中节点间的连接总线状态，可以为 `connected` 或者 `disconnected`
- slot 一个哈希 slot 数字或者区间，从 9 开始，最大为 16384. 这是每个节点各自维护的哈希槽.

## flags

- myself 当前连接的节点
- master 这个节点为 master 节点
- slave slave 节点
- fail? 节点处于 `PFAIL` 状态，对于当前连接的节点不可达，但是逻辑上可达
- fail 节点处于 `FAIL` 状态，对于多个节点都不可达，从 `PFAIL` 状态转来
- handshake 不可信任的节点，当前正在握手
- noaddr 这个节点的地址不可知
- noflags
