---
title: 从 example 学习 kite 源码
category: 后端
tags: [golang, kite]
date: 2019-01-20 14:00
---

使用 kite 生成服务的大致步骤可分为如下

- 编写 thrift 服务描述文件，生成对应代码
- 使用 thrfit 生成的接口，实现 server, client 端的逻辑
- 编译运行，发起 RPC 请求

现在从一个 example 中学习 kite 的具体使用

# thrift IDL 描述文件

## base 描述文件

这个文件描述了通用的 request 和 response 请求

```thrift
// base.thrift
namespace py base
namespace go base
namespace java com.bytedance.thrift.base

struct TrafficEnv {
    1: bool Open = false,
    2: string Env = "",
}

struct Base {
    1: string LogID = "",
    2: string Caller = "",
    3: string Addr = "",
    4: string Client = "",
    5: optional TrafficEnv TrafficEnv,
    6: optional map<string, string> Extra,
}

struct BaseResp {
    1: string StatusMessage = "",
    2: i32 StatusCode = 0,
    3: optional map<string, string> Extra,
}
```

以上代码包含以下几部分的声明

- Traffic Env
- Base
  base of the request
- BaseResp
  base of the BaseResq

## example 描述文件

这个文件描述了 example 中具体的 request 和 response

```thrift
include "base.thrift"

namespace go toutiao.kite.example
namespace py example

struct ThriftItem {
    1: i64 ID64
    2: i32 ID32
    3: double NumD
    4: string NameStr
    5: list<i64> List64
    6: list<string> ListStr
    7: map<string, i64> MapStr64
    8: map<string, string> MapStr2
}

struct ExampleRequest {
    1: string Name,
    2: i64 ID64
    3: i32 ID32
    4: double NumD
    5: list<i64> List64
    6: list<string> ListStr
    7: map<string, i64> MapStr64
    8: map<string, string> MapStr2

    255: base.Base Base,
}

struct ExampleResponse {
    1: string Resp,

    255: base.BaseResp BaseResp,
}

service ExampleService {
    ExampleResponse Visit(1: ExampleRequest req)
    ExampleResponse Visit2(1: ExampleRequest req)
}
```

其中主要包含以下几部分

- ThriftItem
- ExampleRequest
- ExampleResponse
- ExampleService

## 生成 golang 代码文件

使用以下命令生成 golang 代码

```shell
kitool new -c -s -i idl/example.thrift -pre org/kite/example -trans Buffered -proto Binary -cmd thrift kite.example
```

# 实现 service 中描述的接口

## server 端实现

以下代码实现了 server 是如何处理 client 发出的对 `Visit` 和 `Visit2` 的请求的

```golang
func Visit(ctx context.Context, r *example.ExampleRequest) (*example.ExampleResponse, error) {
    logs.CtxInfoKvs(ctx, "request", r, "msg", "Handle Visit request")
    name := r.Name + "Visit"
    req2 := &example.ExampleRequest{Base: &base.Base{}, Name: name}
    response, err := ExampleClient.Call("Visit2", ctx, req2)
    if err != nil {
        logs.Error("Call Visit2 error: %s", err)
        return nil, err
    }
    rName := response.RealResponse().(*example.ExampleResponse).Resp
    resp := example.NewExampleResponse()
    resp.Resp = name + Name
    resp.BaseResp = &base.BaseResp{StatusCode: 0}
    return resp, nil
}

func Visit2(ctx context.Context, r *example.ExampleRequest) (*example.ExampleResponse, error) {
    name := r.Name + "Visit2"
    resp := example.NewExampleResponse()
    resp.Resp = Name
    resp.BaseResp = base.NewBaseResp()
    return resp, nil
}
```

## client 端

在 client 实现

```golang
func mkVisit(client *example.ExampleServiceClient) endpoint.Endpoint {
    return func (ctx context.Context, request interface{}) (interface{}, error) {
        transport := client.Transport.(kitc.Transport)
        err := transport.OpenWithContext(ctx)
        if err != nil {
            return nil, err
        }
        defer transport.Close()
        resp, err := client.Visit(requset.(endpoint.KitcCallRequest).RealRequest().(*example.ExampleRequest))
        addr := transport.RemoteAddr()
        return &KitcExampleResponse{resp, addr}, err
    }
}

func mkVisit2(client *example.ExampleServiceClient) endpoint.Endpoint {
    return func(ctx context.Context, request interface{}) (interface{}, error) {
        transport := client.Transport.(kitc.Transport)
        err := transport.OpenWithContext(ctx)
        if err != nil {
            return nil, err
        }
        defer transport.Close()
        resp, err := client.Visit2(request.(endpoint.KitcCallRequest).RealRequest().(*example.ExampleRequest))
        addr := transport.RemoteAddr()
        return &KitcExampleResponse{resp, addr}, err
    }
}

func (c *KitcExampleServiceCaller) Call(name string, request interface{}) (endpoint.Endpoint, endpoint.KitcCallRequest) {
    switch name {
    case "Visit":
        return mkVisit(c.client), &KiteExampleRequest{request.(*exampleRequest)}
    case "Visit2":
        return mkVisit2(c.client), &KiteExampleRequest{request.(*exampleRequest)}
    }
    return nil, nil
}
```

kitool 生成的客户端代码中自动生成了以下接口

- `KitcExampleClient`
- `KitcExampleServiceCaller`
- `KiteExampleRequest`
- `KitcExampleResponse`
- `mkVisit`
- `mkVisit2`

# RPC 调用

生成客户端代码后，可以使用自动生成的 `Visit` 来调用服务。

```golang
import (
    _ "org/example/clients/example"
    "org/kite/example/thrift_gen/example"
    "org/kite/kitc"
)

var（
    client * kitc.KitcClient
    err      error
)

func init() {
    var err error
    // 使用服务名初始化 client
    client, err = kitc.NewClient("example")
    if err != nil {
        /// ... do something
    }
}

func main() {
    req := example.NewExampleRequest()
    // RPC 调用，调用 Visit 接口
    resp, err := client.Call("Visit", ctx, req)
    // or
    client := example.NewExampleServiceClientFactory(useTransport, protocolFactory)
    req : = &example.ExampleRequest{}
    resp, err := client.Visit(req)
}
```
