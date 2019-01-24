---
title: kitc 源码 kitc_client
category: 后端
date: 2019-01-22 12:00
---

kitc(kite 的客户端代码) 中的 kitc_client.go 源码有 500+ 行

# 代码组成

## 接口

源码中包含以下接口

- discoverChangeHandler
- userErrCBChangeHandler
- NewClient
- NewWithThriftClient
- newWithThrfitClient
- SetChain
- initMWChain
- Call
- initRPCInfo
- Name
- instanceCBChangeHandler
- Options
- RemoteConfigs
- ServiceInstances
- RecentEvents
- ServiceCircuitbreaker
- InstanceCircuitbreaker
- pushEvent
- serviceCBChangeHandler
- GetRPCConfig

## 结构

- KitcClient

# KitcClient

## 结构组成

```golang
type KitcClient struct {
    meshMode        bool
    name            string
    opts            *Options
    client          Client

    userErrBreaker  *circuit.Pannel
    serviceBreaker  *circuit.Panel
    instanceBreaker *circuit.Panel
    pool            *connpool.ConnPool
    discoverer      *kitcDiscoverer
    remoteConfiger  *remoteConfiger
    eventQueue      *kitevent.Queue
    loadbalancer    loadbalancer.LoadBalancer

    chain           endpoint.Middleware
    once            sync.Once
}
```

## 成员解读

### client

client 是一个接口，声明如下

```golang
type Client interface {
    New(kc *KitcClient) Caller
}
```

应由 kitool 工具自动生成

# Call

## 源码

```golang
func (kc *KitcClient) Call(method string, ctx context.Context, request interface{}) {
    metricsClient.EmitCounter("kite.requset.throughput", 1, "", nil)

    kc.once.Do(kc.initMWChain)

    if !ServiceMeshMode {
        if kc.opts.LoadBalancer != nil || kc.opts.IDCHashFunc != nil {
            ctx = context.WithValue(ctx, lbkey, request)
        }
    }

    if ServiceMeshMode {
        if consist, ok := kc.opts.Loadbalancer.(*loadbalancer.ConsistBalancer); ok {
            if ringhashKey, err := consist.GetKey(ctx, request); err == nil && len(ringhashKey) > 0 {
                ctx = context.WithValue(ctx, RingHashKeyType(":CH"), ringhashkey)
            }
        }
    }

    ctx, err := kc.infoRPCInfo(method, ctx)
    if err != nil {
        return nil, err
    }

    caller := kc.client.New(kc)
    next, request := caller.Call(method, request)
    if next == nil || request == nil {
        return nil, fmt.Errorf("service=%s method=%s unknow method return nil", kc.name, method)
    }

    ctx = context.WithValue(ctx, KITECLIENTKEY, kc)
    resp, err := kc.chain(next)(ctx, request)
    if _, ok := resp.(endpoint.KitcCallResponse); ! ok {
        return nil, err
    }
    return resp.(endpoint.KitcCallResponse), err
}
```

这部分代码中的具体的实现依赖于 kitool 实现的客户端部分。
该方法的实际调用者为由 thrift 生成的中的继承自 `kitcClient` 的实例, 中间 `caller.Call` 接口是由 thrift 文件中生成的。

这个方法是 client 向 RPCServer 发起 RPC 请求，根据 `method` 参数确认要调用的方法

## 逻辑分析

- 在监控中增加计数
- 初始化 RPC info
- caller 调用方法
- 调用 middleWare
- 返回 response
