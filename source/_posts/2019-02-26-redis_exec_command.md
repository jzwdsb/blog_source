---
title: redis 执行命令的过程
category: 数据库
date: 2019-02-26 12:00
tags: [redis]
---

redis 的在启动后创建了一些循环事件来监听 TCP 端口和 Unix 的 Sockets, 从而使 redis 服务器可以接受新的连接，之后就可接受来自 client 的请求和命令。
中间的过程主要分为以下几步

- [处理新连接](#处理新连接)
- [读一个客户端的命令](#读一个客户端的命令)
- [执行命令并返回](#执行命令并返回)

# 处理新连接

redis 在 initServer() 函数中创建循环事件调用了 `acceptTcpHandler` 和 `acceptUnixHandler` 函数来处理接受到的 TCP 连接和 Unix 的 Sockets 连接。这两个函数又调用了 `acceptCommonHandler`, 在这个函数中又调用了 `createClient` 函数创建了一个新的 client 对象，用来表示一个新的客户端连接

`createClient` 的工作主要如下

首先为变量 c 分配了内存，接着将 Socket 连接置为非阻塞状态，并且设置了 TCP 无延迟。然后创建了 File 循环事件(`asCreateFileEvent`) 来调用 `readQueryFromClient`. 新建的客户端默认连接的是服务器的第一个数据库(编码为 0), 最后设置好客户端的各种属性和状态.

# 读一个客户端的命令

`readQueryFromClient` 函数就是用来从客户端读取命令的，具体实现如下

Redis 会先将命令读入缓冲区，一次最多读取的大小是 `PROTO_IOBUF_LEN`(1024 x 16) bit. 然后调用 `processInputBufferAndReplicate` 函数, 来处理缓冲区中的数据，如果客户端时 master(主从同步过程), 那么 Redis 就会计算前后缓冲区的不同部分，以确定从节点接受了多少数据。
`processInputBufferAndReplicate` 函数会处理客户端向服务器发送命令和主节点向从节点发送命令这两种情况，不过最后都需要调用 `processInputBuffer` 函数.

`processInputBufferAndReplicate` 函数会先判断客户端是否正常，如果出现连接中断或者客户端阻塞等情况，就会立即停止命令。然后根据读取的请求生成 Redis 可以执行党的命令(包括参数). 不同的请求类型分别调用 `processInlineBuffer` 和 `processMultbulkBuffer` 函数，生成好命令之后，交给 `procssCommand` 执行，如果返回 `C_OK` 则重置客户端，等待下一个命令。如果返回的是 `C_ERR`, 客户端就会销毁。(比如执行 `QUIT` 命令

`processCommand` 函数会从 Redis 启动时加载的命令表中查找命令，然后检查命令的执行权限。

如果是 cluster, 这是会判断 key 是否属于当前的 master, 不属于返回重定向信息。

如果内存不够用，这里也需要判断一下是够有可以释放的内存，如果没有，就不能执行命令，返回错误信息。

接下来判断一些不能接受写命令的情况

- 服务器不能进行持久化
- 作为 master, 没有足够的可用的 slave
- 此服务器为只读的 slave, 只有它的 master 可以接受命令

在订阅模式中，只能接受指定的命令

- (P) `SUBSCRIBE`
- (P) `UNSUBSCRIBE`
- `PING`
- `QUIT`

当 slave 和 master 失联时，只能接受有 flag `t` 的命令，例如 `INFO`, `SLAVEOF`

如果命令没有 `CMD_LOADING` 标志，并且当前服务器正在加载数据，则不能接受此命令

对 lua 脚本长度进行限制

当进行完上述的各种条件判断后，才真正开始调用 `call` 函数执行命令

# 执行命令并返回

`call` 函数的参数是 client 类型，取出 cmd 成员进行执行.

```cpp
dirty = server.dirty;
start = ustime();
c->cmd->proc(c);
duration = ustime() - start;
dirty = server.dirty - dirty;
if (dirty < 0) dirty = 0;
```

如果是写命令，就是在服务器上产生脏数据，服务器需要标记下内存中的某些有了改变。这对于 Redis 的持久化来说非常重要，它可以知道这个命令影响了多少个 key. 命令执行完毕后并没有结束，call 函数还会进行一些其他操作。例如记录日志，写 AOF 文件，向从节点同步命令等。