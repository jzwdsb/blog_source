---
title: golang 中的 gracefull shutdown
category: 后端
date: 2019-02-11
tags: [golang]
---

在 golang 1.8 之后，对于支持的 http 服务支持 gracefull shutdown, 相比起之前的实现，golang 中的 server 在退出后没有做资源清理，在实现 gracefull shutdown 之后，只需要调用 shutdown 就可以实现

```golang
func (srv *Server) Shutdown(ctx context.Context) error
```

`Shutdown` 将无中断的关闭正在活跃的连接，然后平滑的停止服务。处理流程如下

- 首先关闭所有的监听
- 然后关闭所有的空闲连接
- 然后等待无限期等待连接处理完毕转为空闲，并关闭
- 如果提供了带有超时的 `Context`, 将在服务关闭前返回 `Context` 的超时错误

这样在运行 server 时，`ctrl + c` 后，server 会先释放其所占有的资源，比如使用的端口号，占用的连接，然后才会关闭。而在之前，关闭后这些资源仍然占用，直到 OS 将其释放。