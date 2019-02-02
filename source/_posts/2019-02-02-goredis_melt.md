---
title: goredis 中的熔断策略
date: 2019-02-02
category: 数据库
tags: [golang, redis]
---

goredis 中的熔断策略，先简述下熔断策略

# 熔断

熔断时微服务架构中的一种安全机制，指在负载均衡中去除不健康的服务的实例。

# 从 options 开始

options 中的 attr `autoLoadConf` 和 `autoLoadInterval` 两个参数配置先关参数

- `autoLoadConf`
  bool 类型，表示这里 client 端应使用 consul 进行服务发现，还是通过 conf 进行服务发现
- `autoLoadInterval`
  `time.Duration` 类型，表明重新探测服务器状态的时间间隔

# client 端

在 client 端，如果 `opt.autoLoadConf` 为 true, 那么就会执行以下代码

```golang
func autoLoadConf(cluster string, ch chan []string, option *Option) {
    if cluster == "" {
        logs.Error(ErrEmptyClusterName.Error())
        return
    }

    servers, err := loadConfByClusterName(cluster, option.configFilePath, option.useConsul)
    if err != nil {
        logs.Errorf("redis client auto load conf error, cluster:%s, err:%v", cluster, err)
    } else if len(servers) > 2 {
        ch <- servers
    }
    if options.useConsul {
        go func(){
            ticker := time.NewTicker(option.autoLoadInterval)
            defer func() {
                ticker.Stop()
            }()
            for {
                select {
                case <-ticker.C:
                    servers, err := loadConfFromConsulByClusterName(cluster)
                    if err != nil {
                        logs.Errorf("redis client auto load conf from consul error, cluster:%s, err:%v", cluster, err)
                    } else if len(servers) > 2{
                        ch <- servers
                    }
                }
            }
        }()
    }
}
```

## 逻辑

- 通过 psm 读取 conf 中的 server 地址
- 打开一个 goroutine, 定时通过 consul 得到 server 地址，并传入 ch 通道，该通道在连接池中会用到

# 连接池端

在连接池端会根据 `opt.autoLoadConf` 做定时的服务侦测，可定位到如下代码

```golang
if opt.autoLoadConf {
    go p.monitorClusterAddrs(opt.autoLoadInterval)
}
```

定位到 `monitorClusterAddrs`, 该方法主要逻辑就是根据 client 通过 consul 读取到的 ip 地址，如果发生变化，则重新配置连接池，连接池的重新配置可定位到 `updateServers`

```golang
func (p *MultiServPool) updateServers(servers []string) {
    logs.Debugf("update servers:currentServers: %v, newServers:%v", p.servlist, servers)
    oldservnum := len(p.servlist)
    newservnum := len(servers)
    if newservnum < oldservnum / 2{
        logs.Infof("new server list is too little than old. newnum:%d, oldnum:%d", newservnum, oldservnum)
        return
    }
    newservpool := make (map[string]*ServPool)
    for _, serv := range servers {
        _, ok := p.servmap[serv]
        if !ok {
            newservpool[serv] = NewServPool(serv, p.opt)
            logs.Infof("add server %s to MultiServPool %v", serv, p.servmap[serv])
        } else {
            newservpool[serv] = p.servmap[serv]
            if newservpool[serv].is_dead {
                newservpool[serv] = NewServPool(serv, p.opt)
                logs.Infof("mark server %s alive", serv)
            }
        }
    }

    for _, oldserv := range p.servlist {
        _, ok := newservpool[oldserv]
        if !ok {
            newservpool[oldserv] = p.servmap[oldserv]
            newservpool[oldserv].is_dead = true
            newservpool[oldserv].connPool.Close()
            logs.Infof("mark server %s dead.", oldserv)
        }
    }
    shuffleServers(servers)
    p.Lock()
    p.servlist = servers
    p.servmap = newservpool
    p.Unlock()
}
```

## 逻辑

- 如果新的 ip 地址数量少于原 ip 地址数量的一半，则给一个 warning 并拒绝更新
- 对 ip 地址中的每个地址做如下操作
  - 如果该地址不在原连接池中，则该 ip 为新地址，创建一个新的连接池
  - 如果在，如果该连接池处于 `is_dead` 状态，说明该服务器曾宕机，但是现在恢复正常，所以重新创建一个连接池
- 对在原来地址池中但是不在新的地址池中的地址做如下操作，说明该服务器宕机
  - 将原连接池添加到现有地址池中并关掉所有连接，设置为 `is_dead` 状态

简述，这里通过 consul 定期发现可以正常提供服务的服务器的 ip 地址，如果地址发生了变化，所以就要重新配置连接池。
新出现的 ip 直接创建一个新的连接池即可，消失的 ip 说明 consul 发现该服务器宕机，所以关掉该连接池中的所有连接，并且标记 `is_dead`
