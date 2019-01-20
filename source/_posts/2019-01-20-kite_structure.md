---
title: kite 架构
category: 后端
date: 2019-01-20 20:00
tags: [golang, kite]
---

kite 分为服务端框架，客户端框架和相关的组件

- 服务端: `kite`
  提供了创建一个服务的解决方案
- 客户端框架 `kitc`
  提供了 RPC 调用的方案，并整合了服务发现，连接管理等功能

![kite_structure](/image/kite_structure.png)

使用 kite 开发的服务，其主要结构如上图所示。请求到达后，由 kite 框架进行处理，然后交由对应的 handler 进行处理，handler 内部需要执行 RPC 调用的时候，向下穿透进入到 kitc 框架。

# EndPoint

`EndPoint` 是抽象接口层，由 kitool 工具自动生成，用语 Thrift 服务入口层和业务逻辑层。

# Middleware

`Middleware` 用于对 `EndPoint` 进行封装，实现对应的功能，由于 `Middleware` 本身是可组合的，这样就将微服务架构中的多种功能组件化，例如 `Metrics` 发送，访问控制等，不需要将这些功能耦合到具体的业务逻辑中。

在 kitc 框架中，抽象接口层的上下两层也都是 kitool 生成的代码，通过实现不同的 Middleware, 实现不同的 Middleware, 实现服务降级，断路器，机房流量调度等容灾相关的功能。

# Client

kitc 的 Client 接口实现了 RPC 调用的服务发现和连接管理功能，具体架构如下

![kitc_client](/image/kitc_client.png)

# ConnPool

在 client 的层次设计中，把协议和网络连接本身首相。网络连接层被抽象为 `ConnPool` 接口，只有两个方法

```golang
type ConnPool interface {
    Get() (net.Conn, error)
    Put(c net.Conn, err error) error
}
```

框架在实现 `ConnPool` 的采用了两种方式进行实现，分别为短连接的实现和长连接的实现，但这些对上层透明。

`Transport` 和 `Protocol` 主要是针对 Thrift 定义的，kite 框架服务默认使用统一的传输层和协议层，分别是 `BufferdTransport` 和 `BinaryProtocol`.

# IService

对服务发现的一个标准定义，仅有两个方法

```golang
type IService interface {
    Name() string
    Lookup(dc string) []* Endpoint
}
```

给定一个 IService 的实现，可以根据其方法获取服务名和指定 IDC 的服务地址列表。其标准的实现是从中获取具体指定 IDC 的 IDC 的具体服务的地址列表。