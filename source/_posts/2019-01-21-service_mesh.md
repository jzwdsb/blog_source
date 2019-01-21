---
title: Service Mesh 相关
date: 2019-01-21 15:00
category: 后端
---

Service Mesh(服务网格), 是一种微服务架构的形式。

# 维基中的定义

在 wiki 中解释如下

> In a service mesh, each service instance is paired with an instance of a reverse proxy server, called a service proxy, sidecar proxy, or sidecar. The service instance and sidecar proxy share a container, and the containers are managed by a container orchestration tool such as Kubernetes. The service proxies are responsible for communication with other service instances and can support capabilities such as service (instance) discovery, load balancing, authentication and authorization, secure communications, and others.
> In a service mesh, the service instances and their sidecar proxy are said to make up the data plane, which includes not only data management but also request processing and response. The service mesh also includes a control plane for managing the interaction between services, mediated by their sidecar proxies. There are several options for service mesh architecture: Istio (a joint project among Google, IBM, and Lyft), Buoyant[31] & others

在 Service Mesh 中，每个微服务的实例与一个反向代理服务器绑定，被称为 service proxy, sidecar proxy 或者 sidecar. service instance 和 sidecar proxy 共享一个容器，此容器由一个容器编排工具(比如 Kubernetes) 管理。这些 service proxy 负责与其他 service instance 通信，并且具有以下能力

- 服务发现
- 负载均衡
- 访问控制和鉴权
- 安全通信

在 Service Mesh 中，service instance 和对应的 service proxy 组成 data plane, 这不仅包含了数据管理同样也包含了处理请求和发送响应。Service Mesh 同样包括一个控制面板来控制服务间的交互，并由 sidecar proxy 调解.

目前较为通用的 Service Mesh 架构有 Istio, Linkerd 等

# 较为通俗的解释

> A service mesh is a configurable infrasturcture layer for a microservices application.

Service mesh 是微服务应用中可配置的基础设施层，可以理解为 TCP/IP 之上的抽象层。

Service Mesh 有如下几个特点：

1. 应用程序间通讯的中间层
2. 轻量级网络代理
3. 应用程序无感知
4. 解耦应用程序的重试，超时，监控，追踪和服务发现
