---
title: kite 源码分析-- kite_server
category: 后端
tags: [golang, kite]
---

kite_server 源码只有 300+ 行，现对其进行逐模块的解读

# 代码组成

`kite_server.go` 文件主要声明了以下接口

## 公共接口

- `Listen`
- `AcceptLoop`
- `Serve`
- `Stop`
- `PushEvent`
- `RemoteConfigs`
- `OverLoad`
- `RecentEvents`
- `NewPrcServer`

声明的结构

- `RpcServer`

## 私有接口

- `get_RPCConfig`
- `report`
- `processRequests`
- `onPanic`

# RpcServer 结构

RpcServer 提供了 RPC 服务端功能

```golang
type RpcServer struct {
    l net.Listener

    processorFactory thrift.TProcessorFactory
    transportFactory thrift.TTransportFactory
    protocolFactory  thrift.TProtocolFactory

    remoteConfiger *remoteConfiger
    overloader     *overloader
    eventqueue     *kitevent.Queue // store latest events
    reporter       kiteReporter

    lastPanicReport int64
    lastConnReport  int64
    lastQPSReport   int64
}
```

## net.Listener 成员

net 包提供了网络 I/O 接口，包括 TCP/IP, UDP, 域名解析和 Unix 域套接字。
而 `net.Listener` 则提供了面向流协议的通用的网络监听器。
接口如下

```golang
type Listener interface {
    Accept() (Conn, error)
    Close() error
    Addr()  Addr
}
```

## thrift 成员

- `thrift.TProcessorFactory`
- `thrift.TTransportFactory`
- `thrift.TProtocolFactory`

源于 Thrift 中的工厂方法，用于生成 `TProcessor`, `TTransport`, `TProtocol` 对象

## 各种适配器

- `remoteConfiger`
- `overloader`
- `kitevent.Queue`
- `kiteReport`

## 最后一次事件

均为 `int64` 对象

- `lastPanicReport`
- `lastConnReport`
- `lastQPSReport`

# Listen

## Listen 源码

```golang
func (p *RpcServer) Listen() error {
    var addr string
    if ServiceMeshMode {
        addr = ServiceMeshIngressAddr
        if _, err := net.ResolveIPAddr("tcp", addr); err == nil {
            ListenType = LISTEN_TYPE_TCP
        } else {
            ListenType = LISTEN_TYPE_UNIX
            syscall.Unlink(addr)
        }
    } else {
        if ListenType == LISTEN_TYPE_TCP {
            addr = ServiceAddr + ServicePort
        } else if ListenType == LISTEN_TYPE_UNIX {
            addr = ServiceAddr
            syscall.Unlink(ServiceAddr)
        } else {
            return errors.New(fmt.Sprintf("Invalid listen type %s", ListenType))
        }
    }

    logs.Info("KITE: server listen on %s", addr)
    l, err := net.Listen(ListenType, addr)
    if err != nil {
        return err
    }
    p.l = l

    if ListenType == LISTEN_TYPE_UNIX {
        os.Chmod(ServiceAddr, os.ModePerm)
    }

    return nil
}
```

## Listen 分析

从源码可知 `Listen` 接口的逻辑如下

- 根据 `ServiceMeshMode` 推断 `ListenType`
- 根据 `ListenType` 推断 `addr`
- 调用 `net.Listen`, 返回一个 `Listener` 接口
  调用 `Accept` 方法(阻塞,直到接受到一个请求)可以得到 `Conn` 接口，进行通信
- 如果监听模式为 Unix 套接字模式
  - 调用 `os.Chmod` 改变文件权限

# AcceptLoop

## AcceptLoop 源码

```golang
func (p *RpcServer) AcceptLoop() error {
    for {
        // if l.Close() is called will return closed error
        conn, err := p.l.Accept()
        if err != nil {
            deadline := time.After(ExitWaitTime)
            for {
                select {
                    case <- deadline:
                        return err
                    default :
                    if p.overloader.ConnNow() == 0 {
                        return nil
                    }
                    time.Sleep(time.Millsecond)
                }
            }
        }

        if !ServiceMeshMode {
            if ! p.overloader.TakeConn() {
                msg := fmt.Sprintf("KITE: connection overload, limit=%v, now=%v, remote=%s\n",
                    p.overloader.ConnLimit(), p.overloader.ConnNow(), conn.RemoteAddr().String())
                logs.Warnf(msg)
                p.report(&p.lastConnReport, "Connection Overload", msg)
                conn.Close()
                continue
            }
            if !p.overloader.TakeQPS() {
                msg := fmt.Sprintf("KITE: qps overload, qps_limit=%v, remote=%s\n", p.overloader.QPSLimit(), conn.RemoteAddr().String())
                logs.Warnf(msg)
                conn.Close()
                p.report(&p.lastQPSReport, "QPS Overload", msg)
                p.overloader.ReleaseConn()
                continue
            }
        }

        go func(conn net.Conn) {
            if GetRealIP {
                // encode real ip to stack
                handleRPC := func () {
                    client := thrift.NewTSocketFromConnTimeout(conn, ReadWriteTimeout)
                    if err := p.processRequests(client); err != nil {
                        logs.Warnf("KITE: processing request error=%s, remote=%s", err, conn.RemoteAddr().String())
                    }
                }
                addrCode := encodeAddr(conn.RemoteAddr().String())
                if addrCode == 0 {
                   logs.Debugf("KITE: invalid remote ip: %s", conn.RemoteAddr().String())
                   handleRPC()
                } else {
                    gls.SetGID(addrCode, handleRPC)
                }
            } else {
                client := thrift.NewTSocketFromConnTimeout(conn, ReadWriteTimeout)
                if err := p.processRequests(client); err != nil {
                    logs.Warnf("KITE: processing request error=%s, remote=%s", err, conn.RemoteAddr().String())
                }
            }
       }(conn)
    }
}
```

## AcceptLoop 逻辑分析

代码的主体是一个事件循环, 循环一开始会阻塞在事件开始的第一行代码

```golang
conn, err := p.l.Accept()
```

当就接受到一个请求后，才会继续向下运行

在非服务网格模式下

- 打印当前网络状态
- 关掉当前连接
- 回到事件循环起始位置

可知在服务网格模式下才会执行以下逻辑，忽略打印 log 和调试信息
开启一个 gorountine 处理请求

总结以上逻辑如下

- 进入事件循环，阻塞在等待请求处
- 如果处于非 Service Mesh 模式下
  - 打印日志信息
  - 断开连接
  - 回到 1 处
- 开启一个 goroutine 处理请求

# Serve

## Serve 源码

```golang
func (p *RpcServer) Serve() error {
    if Processor == nil {
        logs.Fatal("KITE: Processor is nil")
        panic("KITE: Processor is nil")
    }

    if err := p.Listen(); err != nil {
        return err
    }
    p.processorFactory = thrift.NewTProcessorFactory(Processor)
    if EnableMetrics {
        GoStatMetrics()
    }
    p.startDaemonRountines()
    startDebugServer()

    if ListenType == LISTEN_TYPE_TCP {
        if err := Register(); err != nil {
            logs.Errorf("KITE: Register service error: %s")
        }
    }
    return p.AcceptLoop()
}
```

## Serve 分析

- 打开 RpcServer 的监听
- 打开 RpcServer 的守护进程
- 打开 DebugServer
- 如果监听模式为 `LISTEN_TYPE_TCP`
  - 注册到 consul 方便查找
- 执行 `AcceptLoop` 处理请求
