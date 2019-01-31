---
title: goredis 中的连接池
category: 数据库
date: 2019-01-31
tags: [goredis, golang]
---

goredis.client 的连接池的实现应该是由其成员 `custerpool *MultiServPool` 实现的。

# MultiServPool 的源码

```golang
type ServOpt struct {
    network     string
    dialTimeout time.Duration
    serv        string
    breaker     *circuit.Breaker
}

type ServPool struct {
    connPool    pool.Pooler
    servopt     *ServOpt
    is_dead     bool
}

type MultiServPool {
    servlist []string
    servmap  map[string]*ServPool
    ch       chan []string
    opt      *Option
    cursor   uint32
    sync.RWMutex
}
```

# 创建连接池

## 源码

```golang
func NewMultiServPool(servers []string, ch chan []string, opt *Option) *MultServPool {
    var nowservers []string
    servmap := make(map[string]*ServPool)
    for _, serv := range servers {
        servslice := strings.Split(serv, ":")
        host := servslice[0]
        port := servslice[1]
        iplist, err := net.LookupHost(host)
        if err != nil {
            logs.Warnf("lookup host %s err. err:%s", host. err)
        } else {
            host = iplist[0]
        }
        newserv := host + ":" + port
        nowservers = append(nowservers, newserv)
        servmap[newserv] = NewServPool(newserv, opt)
    }
    shuffleServers(nowservers)
    logs.Infof("host to ip done. servlist:%v", nowservers)

    p := &MultiServPool {
        servlist: nowservers,
        ch: ch,
        opt: opt,
        cursor: 0,
    }
    p.servmap = servmap

    if opt.autoLoadConf {
        go p.monitorClusterAddrs(opt.autoLoadInterval)
    }

    if opt.IdleCheckFrequency > 0 && opt.IdleTimeout > 0 {
        go p.reaper(opt.IdleCheckFrequency)
    }
    return p
}
```

## 逻辑

- 对于传入的 `servers` 中的每个地址做域名解析
- 保存解析的 ip 地址，添加到 `nowservers` 和  `servmap` 中
- 随机打乱 `nowservers`
- 根据 options 的中的 `autoLoadConf` 和 `IdleCheckFrequency` 分别打开一个协程，每隔给定时间间隔就分别重新更新服务器 ip 列表，关闭空闲连接

# 接口

以下接口是在 goredis.MultiServPool 在 goredis.Client 中常用到的接口

- getConnnection
- releaseConnetion

当 goredis.Client 从连接池中获取一个连接时会调用 `getConnection` 接口，将连接放回到连接池中时会调用 `releaseConnection` 接口.

## getConnection

源码

```golang
func (p *MultiServPool) getConnection() (*pool.Conn, bool, error) {
    server := p.getServer()
    cn, isNew, err := server.connPool.Get()
    if err != nil {
        return nil, false, err
    }
    return cn, isNew, err
}
```

由此追踪到两个主要方法，`MultiServPool.getServer` 和 `redis.pool.Pooler.Get`, 当然这两个方法在代码中是依赖的，后者依赖前者的调用。

`getServer` 的逻辑是从当前 `servermap` 中选取游标所指的 server, 如果该 server 工作正常且断路器正常工作，就返回该 server, 如果所有都不正常工作，就随机选择一个并给出一个 warning.

具体获得底层连接是通过 `Pooler.Get` 得到的。

### Get

```golang
func (p *ConnPool) Get() (*Conn, bool, error) {
    if p.closed() {
        return nil, flase, ErrClosed
    }

    atomic.AddUint32(&p.stat.Requests, 1)
    select {
    case p.queue <- struct{}{}:
    default :
        timer := timer.Get().(*time.Timer)
        timer.ReSet(p.opt.PoolTimeout)

        select {
        case p.queue <- struct{}{}:
            if !timer.Stop() {
                <- timer.C
            }
            timers.Put(timer)
        case <- timer.C:
            timers.Put(timer)
            atomic.AddUint32(&p.stats.Timeouts, 1)
            return nil, false, ErrPoolTimeout
        }
    }

    for {
        p.freeConnsMu.Lock()
        cn := p.popFree()
        p.freeConnsMu.UnLock()

        if cn == nil {
            break
        }

        if cn.IsStable(p.opt.IdleTimeout, p.opt.LiveTimeout) {
            p.CloseConn(cn)
            continue
        }
        atomic.AddUint32(&p.stats.Hits, 1)
        return cn ,false, nil
    }

    newcn, err := p.NewConn()
    if err != nil {
        <- p.queue
        return nil, false, err
    }
    return newcn, true, nil
}
```

该部分主要逻辑是从现有的连接池中取出一个连接或者创建新连接，仅在没有空闲连接时才会创建新连接。
中间用了 chan 和锁来保证并发安全性。

# 空闲连接的管理

这里空闲连接的控制主要是指 opt.IdleCheckFrequency > 0 时调用的 reaper 协程

## 源码

```golang
func (p *MultiServpool) reaper(frequency time.Duration) {
    ticker := time.NewTicker(frequency)
    defer ticker.Stop()

    for range ticker.C {
        for serv, pl := range p.servmap {
            n, err := pl.connPool.(*pool.ConnPool).ReapStaleConns()
            if err != nil {
                logs.Info("ReapStaleConns failed: %s serv: %s", err, serv)
                continue
            }
            s := pl.connPool.Stats()
            if n > 0 {
                logs.Info("reaper: removed %d stale conns (TotalConns=%s FreeConns=%d Requests=%d Hits=%d Timeout=%d serv=%s)", n, s.TotalConns, s.FreeConns, s.Requests, s.Hits, s.Timeouts, serv)
            }
        }
    }
}
```

这段代码每隔 frequency 时间间隔就会执行一次，使用 `ServePool.ConnPool.ReapStaleConns` 清理空闲连接。这个方法又会调用 `reapStaleConn` 来清理空闲连接

```golang
func (p *ConnPool) reapStaleConn() bool {
    if len(p.freeConns) == 0 {
        return false
    }

    cn := p.freeConns[0]
    if !cn.IsStale(p.opt.IdleTimeout, p.opt.LiveTimeout) {
        return false
    }

    p.CloseConn(cn)
    p.freeConns = append(p.freeConns[:0], p.freeConns[1:]...)
    return true
}
```

简述空闲连接清理逻辑

- options 配置 IdleCheckFrequency 空闲连接清理频率
- 开启一个协程，调用 reaper, 每隔一定时间间隔就开始清理
- 空闲连接缓存在 `p.freeConn` 中，直接取第一个 conn, 如果该连接不处于 stale 状态，则该连接不能清理，所以可以直接返回 false
- 关闭该连接，将该连接从 Slice 中清除

## 关于 p.freeConn

以上清理空闲连接实际上是直接清理 `p.freeConn` 中的第一个，这里 p 中的连接排布就可能影响到到不同情况下的清理策略，因此需要详细看下。

`p.freeConn` 的 appen 和 del 发生在以下几个方法中

- `popFree`
- `Put`
- `reapStaleConn`

上面的对空闲连接的管理对应到 client 层次分别在以下几个场景中被调用

- client 调用 `ReleaseConn` 向连接池返还连接
- client 调用 `GetConn` 从连接池中获取连接
- reaper 所在的 `goroutine` 定时调用

当返还时直接添加到 `p.freeConn` 的尾部，`popFree` 也是从尾部取出空闲连接，当 `reaper` 时则是从头部删除空闲连接

由于 client 返还并无一定的顺序性，所以这里添加删除策略也是比较简单，中间连接的顺序应该也是业务无关的。