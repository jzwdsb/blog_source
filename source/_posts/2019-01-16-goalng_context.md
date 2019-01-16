---
title: golang 中的 context
category: golang
date: 2019-01-16 10:00
---

# 背景

在 Go server 中，每个刚收到的 request 都由对应的 goroutine 处理。Requests handlers 经常开启一个额外的 goroutines 去访问后端，比如数据库或者 RPC 服务。
工作在特定 Request 上的 goroutine 集合通常需要访问 Request 指定的数据，比如终端用户的身份，认证秘钥，以及最后期限。
当一个 Request 取消后或者 time out(过期), 服务于这个 Request 的 goroutine 都需要迅速退出，这样系统可以回收他们正在使用的资源

go 中开发了 `context` 包来方便地在处理同一个 Request, 跨越 API 边界的 goroutine 的集合中传递 数据，取消信号和最后期限。

# Context

Context 接口定义

```golang
DeadLine() (deadline time.Time, ok book)
Done () <- chan struct {}
Err () error
Value (Key interface{}) interface {}
```

## Deadline

```golang
`Deadlin(）(deadline time.Time, ok bool`
```

返回当前任务应当结束的时间，同样也是这个 context 应当取消的时间
ok == false 表示没有设置 deadline. 对 `Deadline` 的连续的成功的调用返回相同的值

## Done

```golang
`Done() <- chan struct{}`
```

返回一个 closed 的 channel. 当任务已经结束，表示当前的 context 的应当被结束。
`Done` 可能返回 nil 表示当前 context 没有永远不能取消。对 `Done` 成功的调用应当返回同样的值。
`Done` 用语 `select` 语句。
`WithCancel` 当 `cancel` 发生时将 `Done` 为 `closed`
`WithDeadline` 当 `deadline` 到了后将 `Done` 设置为 `closed`
`WithTimeout` 当 `timeout` 时间计数过期后将 `Done` 设置为 `closed`

```golang
    func Stream(ctx context.Context, out chan <- Value) error {
        for {
            v, err := DoSomething(ctx)
            if err != nil {
                return err
            }
            select {
            case <- ctx.Done():
                return ctx.Err()
            case out <- v:
            }
        }
    }
```

## Err

```golang
`Err() error`
```

如果 `Done` 尚未设置为 `closed`, `Err` 将返回 `nil`
如果 `Done` 已经 `closed`, `Err` 将返回 `Non-nil` error 说明原因
`Canceled` 表示 context 已经 canceled
`DeadlineExceeded` 表示 context 的 `deadline` 已经过期
当 `Err` 返回 `non-nil` 错误后，对 `Err` 成功的调用都应返回同样的值

## Value

```golang
Value(Key interface{}) interface{}
```

该接口返回在的 `context` 与 `key` 相关联的 `value`, 当没有对应的 `key` 则返回 `value`
对 `Value` 成功的调用相同的 `key` 下应返回相同的 `value`

应仅在跨进程和 API 边界传递 request scope 的数据时使用 context value.

```golang
package user

import "context"

type User struct {...}

type key int

var userKey key

// returns a new Context that carries value u
func newContext(ctx context.Context, u *User) context.Context {
    return context.WithValue(ctx, userKey, u)
}

// returns the User Value stored in ctx, if any
func fromContext(ctx context.Context)(*User, bool) {
    u, ok := ctx.Value(userKey).(*User)
    return u, ok
}
```

# Derived contexts

`context` 包支持从已有的 `context` 中创建新的 `context`. 这些继承关系为一个树结构，且当父 `context` 状态变为 `canceled` 时，继承自该 `context` 的 `context` 也会变为 `canceled`.

所有 `Context` 树的根节点均为 `Background`, 且状态永远不会变为 `canceled`

```golang
func Background() Context
```

## WithCancel 和 WithTimeout

`WithCancel` 和 `WithTimeout` 返回继承自参数的 `context`, 他会比父 `context` 更早的变为 `canceled`.
与到来的 Request 相关联的 `context` 通常会在 request handler 结束时变为 `canceled`.
`WithCancel` 通常用于当使用多个 replicas 时终止多余的 request.
`WithCancel` 通常用于对后端的 request 设置 deadline

```golang
func WithCancel(parent Context)(ctx Context, cancel CancelFunc)

type CancelFunc func()

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

## WithValue

`WithValue` 提供了一个将 request-scopeed 数据与 context 关联的方式

```golang
func WithValue(parent Context, key interface{}, value interface{}) Context
```