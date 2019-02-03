---
title: redis 中的 pipeline 与 mget 比较
category: 数据库
date: 2019-02-02
---

简述下 redis 中 pipeline 和 mget 的区别和优劣

# pipeline

pipeline 是将多个命令一次性写入到服务器，然后等待服务器返回结果，他可以同时执行多个命令，但是各个命令之间不能有数据依赖，因为 pipeline 执行命令式不能保证执行顺序与写入时的命令顺序相同。

因为 redis 中命令是基于 request / response 模式，所以当要连续执行多个命令时，主要的时延在于 RTT, 而且还要频繁调用系统 i/o 接口，所以就有了 pipeline 将多个命令同时一次性写到服务器。

## 适用场景

需要多个命令及时提交，而且命令之间没有数据依赖的时候，适合用 pipeline, 可以较大的提升性能

## 注意

- pipeline 独占一个连接，所以管道内命令太多，可能会请求超时
- 可发送命令数量受 client 端缓冲区大小限制
- redis server 存在 query buffer 限制
- redis server 存在 output buffer 限制
- 需要 server 和 client 的共同支持才能实现 pipeline
- redis cluster 不建议使用 pipeline, 容易产生 max redirect 错误
- twem proxy 可以支持 pipeline

# mget 和 mset

```shell
mset a "a1" b "b1" c "c1"
mget a b c
```

mget 和 mset 也是为了减少网络连接和传输时间设置的，而且当 key 的数目较多时， mget 和 mset 性能要高于 pipeline, 但是 mget 和 mset 也仅限与同时对多个 key 进行操作。

