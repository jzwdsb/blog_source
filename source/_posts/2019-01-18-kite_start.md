---
title: kite 微服务框架初探
category: 后端
date: 2019-01-18 17:00
---

kite 是使用 golang 实现的 RPC 框架，自动集成了微服务所需要的一些功能，例如

- 统一的 RPC 调用(目前支持 thrift)
- 自动服务注册以及发现
- 动态配置
- 熔断
- 限流降级
- RPC 日志
- 自动打一些监控指标
- 过载保护
  
# 简单使用

kite 提供了 kitool 工具，来帮助生成代码和升级工具
总的来说只需要三步就能实现完整的 RPC 过程

- 准备 thrift 文件，用来描述提供的接口
- 生成 server 代码，编写逻辑
- 生成 client 代码，发起 RPC

# 示例

## thrift 文件

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

## kitool 生成代码

```shell
kitool new -c -s -i idl/example.thrift -pre demo/thrift_gen  -trans Buffered -proto Binary -cmd thrift example
```

## 实现其 RPC 的逻辑

客户端

```golang
package main

import (
    _ "kite/example"
    "kite/kitc"
    "context"
    "base"
    "example"
    "time"
)

var (
    ExampleClient *kitc.KitcClient
)

func init() {
    var err error
    ExampleClient, err = kitc.NewClient("toutiao.kite.example",
        kitc.WithTimeout(time.Millisecond*10),
        kitc.WithInstances(kitc.NewInstance("127.0.0.1", "8021", nil)))

    if err != nil {
        logs.Error("NewExampleServiceClient error: %S", err)
        panic(err)
    }
}

// implement visit method
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
    resp.Resp = name + rName
    resp.BaseResp = &base.BaseResp{StatusCode: 0}
    return resp, nil
}

func Visit2(ctx context.Context, r *example.ExampleRequest) (*example.ExampleResponse, error) {
    name := r.Name + "Visit2"
    resp := example.NewExampleResponse()
    resp.Resp = name
    resp.BaseResp = base.NewBaseResp()
    return resp, nil
}
```

服务端实现

```golang
package main

import (
    "logs"
    "kite"
)

func main() {
    kite.Init()
    logs.Info("Server version: %s", kite.ServiceVersion)
    logs.Fatal("%s", kite.Run())
    logs.Stop()
}
```