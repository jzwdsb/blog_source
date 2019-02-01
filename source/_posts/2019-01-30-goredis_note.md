---
title: goredis 源码
date: 2019-01-30 16:00
category: 数据库
tags: [golang, redis]
---

从 goredis 的使用开始入手

创建 redis 的客户端需要先指定 options, redis 的客户端本质上基于 TCP 协议向 redis server 传输符合 redis 协议的命令请求，并根据 redis 协议解析 server 端的返回值。

# options

## 创建

创建 options 可以通过以下接口

- `NewOption`
- `NewOptionWithTimeout`

`NewOption` 使用默认参数初始化 options, 默认值定义在 `option.go` 中。

`NewOptionWithTimeout` 需要以下 7 个参数, 如果参数为 0, 则设置为默认值

- dialTimeout 连接超时
- readTimeout 读超时
- writeTimeout 写超时
- poolTimeout 一个请求从连接池中等待连接的超时
- idleTimeout 连接空闲时间
- liveTimeout 连接存活时间
- poolSize 连接池中的最大个数

## 接口

- DisableAutoLoadConf 禁止自动 load redis 集群代理的服务器地址，默认打开
- SetAutoLoadInterval 自动 load redis 集群代理的时间间隔，默认 30s
- SetCircuitBreakerParam 配置断路器，错误率，时间窗口，默认 60%, 10s
- SetServiceDiscoveryWithConsul 设置只会从 consul 获取 servers, 否则优先从 ss_conf 获取 servers, 若没有再从 consul 获取

## 使用

通常由 `NewClientWithOption` 接受 option 参数

# client

goredis 中的 client 继承自 redis 中的 client, 而 client 实现了 cmdable 接口，因此 goredis.Client 可以直接使用 redis 中的各个命令

## 创建

可以使用以下三个接口创建 client

- `NewClient`
- `NewClientWithOption`
- `NewClientWithServers`

以上三个函数第一个参数都是 cluster, 该参数指定 client 要连接的 abase 数据库，该参数会根据配置自动解析为服务器的 ip.

`NewClientWithOption` 接受一个 option, 创建符合的 client.
`NewClientWithServers` 接受一个 server list 和 一个 option 作为参数. server list 指定 client 的 cluster 的 ip 池。

## 接口

在 goredis 中除了继承自 redis.client 的接口之外，还实现了以下方法

- WithContext 使用给定 context 创建 client
- GetConn
- ReleaseConn 释放连接，放回到连接池
- metricsWrapProcess
- NewPipeline 返回给定命名的 pipeline
- Cluster
- PSM
- MetricsServiceName

其他都是返回内部状态

## 使用

当将其作为 redis 的客户端时就直接使用 client 继承自 redis.Cmdable 的接口, 之后操作 redis 数据库。

接口包含如下, 仅写下常用到接口声明

```golang
type Cmdable interface {
    Get(key string) *StringCmd
    MGet(key string) *StringCmd
    Set(key string, value interface{}, expiration time.Durtaion) *StatusCmd
    Del(keys...string)*IntCmd
}
```

# 一个请求的实现

以 Cmdable.Get 为例

```golang
func (c *cmdable) Get(key string) *StringCmd {
    cmd := NewStringCmd("get", key)
    c.process(cmd)
    return cmd
}
```

调用了 `NewStringCmd` 和 `c.process` 方法
`NewStringCmd` 使用给定参数构造 StringCmd

重点在于 `c.process`

```golang
type cmdable struct {
    process func(cmd Cmder) error
}
```

其中的 process 可向上追踪到 redis.baseClient.Process 中，这是 client 默认处理函数

```golang
func (c *baseClient) defaultProcess(cmd Cmder) error {
    for attempt := 0; attempt <= c.opt.MaxRetries: attempt++ {
        if attempt > 0 {
            time.Sleep(c.retryBackoff(attempt))
        }

        cn, _, err := c.getConn()
        if err != nil {
            cmd.SetErr(err)
            if pkg.IsRetryableError(err){
                continue
            }
            return err
        }

        cn.SetWriteTimeout(c.opt.WriteTimeout)
        if err != writeCmd(cn, cmd); err != nil {
            c.releaseConn(cn, err)
            cmd.SetErr(err)
            if pkg.IsRetryableError(err) {
                continue
            }
            return err
        }
        cn.SetReadTimeout(c.cmdTimeout(cmd))
        err = cmd.readReply(cn)
        c.releaseConn(cn, err)
        if err != nil && pkg.IsRetryableError(err) {
            continue
        }
        return err
    }
    return cmd.Err()
}
```

一个请求的处理如下

1. 当重试次数小于最大重试次数时，重复执行 `2-8`
2. 如果不是第一次，就由重试次数作为 `c.retryBackoff` 参数计算应间隔的时间，并暂停
3. `cn := c.getConn()` 获得当前连接
4. 设置写超时
5. 在连接中写入当前请求，如果出错，则释放链接，并返回 err
6. 设置读超时
7. `cmd.readReply(cn)` 读取服务器的回复
8. 释放链接，如果出错则直接进行下一次尝试
9. 没有出错则直接返回 err
10. 超出最大重试次数，返回 `cmd.Err()`

# cmd

cmd 为各个操作的参数，并携带命令执行的结果，不同的操作有不同的返回值，但是都继承于 basecmd.

- basecmd
- Cmd
- SliceCmd
- StatusCmd
- IntCmd
- DurationCmd
- TimeCmd
- BoolCmd
- StringCmd
- FloatCmd
- StringSliceCmd
- BoolSliceCmd
- StringStringMapCmd
- StringIntMapCmd
- ZSliceCmd
- ScanCmd

大部分要求结果的请求(比如 Get)的 cmd 都会有一个 `Result()` 结果，返回 val, err, val 根据 cmd 类型可有所不同，一般都是返回对应类型，比如 `StringCmd` 返回 `string`, `err`, `IntCmd` 返回一个 `Int`, `err`.

但是写操作(Set, Del等操作)通常会会返回一个 `StatusCmd`, `StatusCmd` 的 `Result()` 返回一个 `string`, `err`, 该 `string` 为本次操作的完整命令，通常只需要 `err` 判断命令是否成功即可。

# PipeLine

goredis 中的 pipeline 继承自 `redis.Pipeliner`

goredis.Pipeline 的源码

```golang
type Pipeline struct {
    redis.Pipeliner
    c                   *Client
    cluster             string
    psm                 string
    metricsServiceName  string
    name                string
}
```

redis.Pipeliner 的声明

```golang
type Pipeliner interface {
    StatefulCmdable
    Process(cmd Cmder) error
    Close() error
    Discard() error
    discard() error
    Exec() ([]Cmder, error)
    Cmds() []Cmder
    SetCmds([]Cmder)
    CmdNum() int
    pipelined(fn func(Pipeliner) error)([]Cmder, error)
}
```

redis.StatefulCmdable 的实现

```golang
type StatefulCmdable interface {
    Cmdable
    Auth(password string) *StatusCmd
    Select(index int) *StatusCmd
    ClientSetName(name string) *BoolCmd
    ReadOnly(name string) *StatusCMd
    ReadWrite() *StatusCmd
}
```

## pipeline 的命令处理函数

每当调用 pipeline 继承自 cmdable 对 redis 操作的接口时，底层的调用的 process 命令处理方法为 pipeline 中重写的 `Process` 方法

```golang
func (c *Pipeline) Process(cmd Cmder) error {
    c.mu.Lock()
    c.cmds = append(c.cmds, cmd)
    c.mu.UnLock()
    return nil
}
```

pipeline 会将每次执行的操作压入一个队列中，直到调用 `Exec`, 将所有请求一起打包发送给服务器，在一个 client-server roundtrip 中处理，服务器每当处理完一个请求就返回，这些请求的执行顺序无法保证，所以 pipeline 中执行的请求之间不要有数据依赖，另外 pipeline 的性能大部分情况下高于 mget, 但是 pipeline 中一次请求太多，延迟就会变长，性能就会降低，所以一次 pipeline 不要包含太多请求。

## Exec

逻辑如下

- 处理中间件请求
- 如果中间件请求存在错误，则将其置为 cmds 的 err
- 如果没有错误则调用 `p.exec()` 执行队列中的命令
- 中间数据计算，失败次数，成功次数
- 处理回应