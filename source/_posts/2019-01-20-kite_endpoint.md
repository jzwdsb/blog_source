---
title: kite 框架中的 endpoint
category: 后端
date: 2019-01-20 15:00
tags: [golang, kite] 
---

这里用到的 endpoint 是 kite 框架中的一部分，这部分定义了 kite 框架的核心接口以及部分通用的接口，kite 和 kitc 都会依赖这里定义的接口。

# 源码

源代码只有一个文件，如下

```golang
type EndPoint func(ctx context.Context, req interface{}) (resp interface{}, err error)

type Middleware func(EndPoint) EndPoint

func Chain(outer Middleware, others ...Middleware) Middleware {
    return func(next EndPoint) EndPoint {
        for i := len(others) - 1; i >= 0; i-- {
            next = others[i](next)
        }
        return outer(next)
    }
}

func Build(mws []Middleware) Middleware {
    if len(mws) == 0 {
        return emptyMiddleware
    }
    return func(next EndPoint) EndPoint {
        return mws[0](Build(mws[1:])(next))
    }
}

func emptyMiddleware(next EndPoint) EndPoint {
    return next
}

type KiteBase interface {
    GetLogID() string
    GetCaller() string
    GetAddr() string
    GetClient() string
    GetEnv() string
    GetCluster() string
}

type KiteBaseExtra interface {
    GetExtra() map[string]string
}

type KiteRequest interface {
    GetBase() KiteBase
    IsSetBase() bool
    RealRequest() interface{}
}

type KiteResponse interface {
    GetBaseResp() KiteBaseResp
    RealResponse() interface{}
}

type KiteBaseResponse interface {
    GetBaseResp() KiteBaseResp
    RemoteAddr() string
    RealResponse() interface{}
}

type KitcCallRequest interface {
    SetBase(kb KiteBase) error
    RealRequest() interface{}
}

type ThriftBase interface {
    GetLogID() string
    GetCaller() string
    GetAddr() string
    GetClient() string
    GetExtra() map[string]string
}
```

# 源码分析

可见源码中声明了如下接口

- EndPoint
- Middleware
- KiteBase
- KiteBaseExtra
- KiteRequest
- KiteResponse
- KiteBaseResp
- KitcCallResponse
- KitcCallRequest
- ThriftBase

如下函数

- Chain
- Build
- emptyMiddleware
  
## 接口说明

### EndPoint

表示从远端调用的一个方法

### Middleware func

消息中间件，处理从 input EndPoint 到 output EndPoint 之间的消息传递

### KiteBase

描述了在一个标准的 request 中的应有的 interface

### KiteBaseExtra

KiteBase 的补充

### KiteRequest

在框架中一个标准的 request 应有的接口

### KiteResponse

在框架中一个标准的 response 应有的接口

### KiteBaseResp

一个 base response 应有的接口

### KitcCallResponse

Kitc 应返回的 Response

### KitcCallRequest

Kitc 中使用 Request

### ThriftBase

我的理解是 Thrift 生成的代码中应有的接口

## 辅助方法

### Chain

连接多个消息中间件，使之多个中间件连接后仍然表现为只有一个 input EndPoint 和一个 output EndPoint

### Build

将一个 array 中的消息中间件连接起来

### EmptyMiddleware

一个空的消息中间件，只是返回其消息中间件