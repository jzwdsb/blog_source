---
title: goredis 中的 pipeline
date: 2019-02-01
category: 数据库
tags: [golang, redis]
---

简述下 goredis 中的 pipeline 的实现，这里 pipeline 发出一个请求到取得结果的具体实现由 client 端侵入式实现，所以具体的实现代码不太好找。

从 `goredis.pipeline` 到 `redis.pipeliner`

# goredis.Pipeline

## 源码

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

## 大致结构和封装

通常从 client 端调用 `Pipeline` 时会得到上面声明的结构，该 `Pipeline` 继承自 `redis.Pipeline`, 在上层调用的接口也大都是对底层 `redis.Pipeline` 的封装。
在这一层调用的接口常为 `Pieline.Exec`, 然而 `Pipeline.Exec` 是对 `Pipeline.exec` 的封装，在这层加了 metrics 信息，所以要从一个请求的实现角度学习的话，这一层也是没必要的。

看下 `Pipeline.exec`, 这一层又是对底层 `redis.Pipeliner.Exec` 的封装，继续向下追踪。

# redis.Pipeliner

## 源码

```golang
type Pipeliner interface {
    StatefunCmdable
    Process(cmd Cmder) error
    Close() error
    Discard() error
    discard() error
    Exec()    error
    Cmds()  []Cmder
    SetCmds([]Cmder)
    CmdNum() int
    pipelined(fn func(Pipeliner) error) ([]Cmderr, error)
}

type Pipeline struct {
    statefulCmdable
    exec pipelineExecer
    mu  sync.Mutex
    cmds    []Cmder
    closed  bool
}
```

## 接口

`Pipeline` 包括如下接口

- Process
- CmdNum
- Cmds
- SetCmds
- Close
- Discard
- discard
- Exec
- pipelined
- Pipelined
- Pipeline

`Exec` 中调用了 `Pipeline.exec` 来做真正命令处理。在这一层找不到 `Pipeline.exec` 何时被设置，于是又开始向上查询到 client 端取得 pipeline 的接口，发现获取 pipeline 时调用的为 `redis.Client.Pipeline`, 并且在这一层定义了 `Pipeline.exec`.
(ノಠ益ಠ)ノ彡┻━┻

# client 中的 pipeline

```golang
func (c *Client) Pipeline() Pipeline {
    pipe := Pipeline {
        exec: c.pipelineExecer(c.pipelineProcessCmds),
    }
    pipe.setProcess(pipe.Process)
    retrun &pipea
}

type pipelineExecer func([]Cmder) error
type pipeLineProcessor func(*pool.Conn, []Cmder) (bool, error)

func (c *baseClient) pipelineExecer(p pipeLineProcessor) pipelineExecer {
    return func(cmds []Cmder) error {
        for attempt := 0; attempt <= c.opt.MaxRetries; attempt++ {
            if attempt > 0 {
                time.Sleep(c.retryBackoff(attempt))
            }
            cn, _, err := c.getConn()
            if err != nil {
                setCmdsErr(cmds, err)
                return err
            }
            canRetry, err := p(cn, cmds)
            ok := c.releaseConn(cn, err)
            if ok {
                break
            }
            if !canRetry || err == nil || !pkg.IsRetryableError(err) {
                break
            }
        }
        return firstCmdsErr(cmds)
    }
}

func (c *baseClient) pipelineProcessCmds(cn *pool.Conn, cmds []Cmder) (bool, error) {
    cn.SetWriteTimeout(c.opt.WriteTimeout)
    if err != writeCmd(cn, cmds...); err != nil {
        setCmdsErr(cmds, err)
        return true, err
    }
    cn.SetReadTimeout(c.opt.ReadTimeout)
    return true, pipelineReadCmds(cn, cmds)
}
```

## 实现逻辑

代码分别设置了 `Pipeline.exec` 和 底层 `statefulCmdable.prcoess`, 所以当使用 pipeline 对 redis 进行操作时比如 Get, Set 时，实际上调用函数如下

```golang
func (c *Pipeline) Process(cmd Cmder) error {
    c.mu.Lock()
    c.cmds = append(c.cmds, cmd)
    c.mu.UnLock()
    return nil
}
```

所以会将要执行的指令压入内部队列中，在调用 `Pipeline.Exec` 时，会传递到 `pipelineProcessCmds`, 由该方法进行实际的操作
`pipelineProcessCmds` 调用 `writeCmd` 将所有命令一次性写入连接中，无法保证服务器执行命令的顺序与传入时相同，所以命令之间不能有数据依赖，然后 `readReply` 接受 server 返回的结果，读取命令的结果顺序与写入时相同。

# 结论简述

使用 pipeline 执行多次命令的逻辑如下

- 每当调用一次操作如 Get, Set 时将命令压入内部队列中，获得 `Cmder`
- 调用 Exec 执行，将命令一次性写入连接，获得每个命令的 err
- 使用 Cmder 获得结果

