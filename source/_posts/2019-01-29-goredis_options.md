---
title: goredis 中的 options
date: 2019-01-29 15:00
category: 数据库
tags: [redis, golang]
---

# goredis 中的 options

其中包含以下成员

- NetWork
  指定使用 tcp 或者 unix, 默认使用 tcp
- Addr
  host:port 地址
- Password
  非必须
- DB
  连接到 server 选择的 DB
- MaxRetries
  重试次数
- MaxRetryBackoff
  重试之间的最大间隔，默认为 512ms
- MinReyryBackoff
  重试之间的最小间隔，默认为 8 ms
- DialTimeout
  建立新连接的等待时间
- ReadTimeout
  套接字读的最长时间，超过认为读操作失败，-1 不设置超时，0 默认
- WriteTimeout
  套接字写的最长时间，超过认为写操作失败
- IdleTimeout
  客户端关闭空闲连接的时间，应少于 server 的超时
- PoolTimeout
  客户端等待连接直到返回一个错误的时间，默认为 readtimeout + 1s
- PoolSize
  打开套接字的最大个数，默认为 runtime.NumCPU * 10
- MinIdleConns
  空闲连接存在的最大个数
- MaxConnAge
  客户端连接的存在最长时间
- IdleCheckFrequency
  空闲连接的检查频率，默认为 1 min, -1 则关闭，但是 idletimeout 超时仍会关闭空闲连接
- TLSConfig
  TLS 配置
- autoloadConf
  自动读取 server conf 文件，默认为 true
- autoLoadInterval
  指定 自动读取 conf 的时间，默认为 10s
- maxFailureRate
- minSample
- windowTime
- configFilePath
- useConsul
- PoolInitSize

# 以下接口

- DisableAutoLoadConf
- SetMaxRetries
  设置 MaxRetries
- SetPoolInitSize
  设置 Pool
- SetAutoLoadInterval
- SetCircuitBreakerParam
- SetConfigFilePath
- SetServiceDiscoveryWithConsul
- SetServiceDiscoveryWithoutConsul
