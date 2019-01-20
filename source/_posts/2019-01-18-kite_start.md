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

# kite 的简单使用

这里为了说明不使用自定义的 thrift 描述文件，直接使用 kite 文件

## 服务端

```golang
package main

import "github.com/koding/kite

func main() {
    k := kite.New("kite", "1.0.0")
    k.Config.Port = 8080
    k.Run()
}
```

代码说明

- `kite.New`  创建一个名字为 `first`, 版本号 `1.0.0` 的 kite.
- k.Config 用于设置 kite 的属性，比如 端口号
- Run 方法表示运行此服务，此方法会阻塞，之后就可以接受请求了

## 客户端

```golang
package main
import (
    "fmt"
    "github.com/koding/kite"
)

func main() {
    k := kite.New("second", "1.0.0")
    client = k.NewClient("http://localhost:6000/kite")
    client.Dial()
    response, _ := client.Tell("kite.ping")
    fmt.Println(response.MustString)
}
```

代码说明

- `k.NewClient`
  创建一个 client, 参数指定要连接的 url
- `Dial`
  client 通过 `Dial` 连接服务器，如果连接失败则返回 error
- `client.Tell`
  该方法的调用会阻塞，直到服务端调用了 callback 函数，之后返回 response
