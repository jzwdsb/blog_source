---
title: redis 的启动过程
date: 2019-02-27 15:00
category: 数据库
tags: [redis]
---

redis 大体上可以分为两部分: 服务端和客户端。观察下服务端启动时都做了下哪些事。

Redis 全部由 C 语言编写，main 函数写在 server.c 中，启动主要分为以下几步

- 初始化全局服务器状态:
- 设置 commend table
- 初始化哨兵模式
- 修复持久化文件
- 处理参数
- initServer
- Shared object
- Shared integers
- 新增循环事件
- 分配数据库
- 监听 TCP 端口
- 初始化 LRU 键池
- Server Cron
- 打开 AOF 文件
- 最大内存限制
- Redis Server 启动
- 从磁盘加载数据
- 最后设置
- 进入循环主事件

# 初始化全局服务器状态

如果 redis-server 命令启动时使用了 `test` 参数，那么就会先进行指定的测试。接下来调用了 `initServerConfig` 函数，这个函数初始化了一个类型为 `redisServer` 的全局变量 Server.
`redisServer` 这个结构包含了非常多的字段，按类别划分，可以分为以下类别。

- General
- Modules
- NetWorking
- RDB / AOF load information
- Faster Pointers to offten lookup command
- Fields used only for stats
- Configuration
- AOF / RDB persistence
- Loggin
- Replicaion
- Synchronous replication
- Limits
- Blocked clients
- Sort parameter
- Zip structure config
- time cache
- Pubsub
- Cluster
- Scripting
- Lazy free
- Assert & bug reporting
- System hardware info

`initServerConfig` 的主要工作是给可以在配置文件中(redis.conf)中配置的变量初始化一个默认值。比较常用的变量有服务器端口号，日志等级等。

# 设置 command table

在 `initServerConfig` 函数，会调用 `populateCommandTable` 函数来设置服务器的命令表，命令表的结构如下

```cpp
struct redisCommand redisCommandTable[] = {
    {"module", moduleCommand, -2, 0, NULL, 0, 0, 0, 0, 0},
    {"get", getCommand, 2, "rF", 0, NULL, 1, 1, 1, 0, 0},
    {"set", setCommand, -3, "wm", 0, NULL, 1, 1, 1, 0, 0},
    {"set", setCommand, -3, "wm", 0, NULL, 1, 1, 1, 0, 0},
    {"setnx", setnxCommand, 3, "wmF", 0, NULL, 1, 1, 1, 0, 0}
}
```

每一项分别代表

- name: 命令的名称
- function: 命令对应的函数句柄
- arity: 命令的参数个数, 如果为 `-N` 表示大于等于 N
- sflags: 命令标志, 标识命令的类型(read/write/admin)
- flags: 位掩码，可由 Redis 根据 sflags 计算
- get_keys_proc: 可选函数，当下面三个不能哪些参数是 key 时使用
- first_key_index: 第一个是 key 的参数
- last_key_index: 最后一个是 key 的参数
- key_step: key 的步长，比如 MSET 的 key_step 是 2, 因为某些命令的参数是 key,val,key,val 这样的形式
- microseconds: 执行命令需要的微秒数
- calls: 该命令被调用总次数

设置好命令表后，redis-server 还会对一些常用的命令设置快速查找方式，直接赋予 server 的成员指针。

```cpp
server.delCommand = lookupCommandByCstring("del");
server.multiCommand = lookupCommandByCString("multi")
```

# 初始化哨兵模式

变量初始化后，就会将启动命令的路径和参数保存起来，方便下次重启的时候使用。如果启动的服务是哨兵模式，那么就会调用 `initSentinelConfig` 和 `initSentinel` 初始化哨兵模式。关于哨兵模式会在另一篇博文里介绍。
`initSentinelConfig` 和 `initSentinel` 都在 `sentinel.c` 文件中。`initSentinelConfig` 函数负责初始化 sentinal 的端口号，以及解除服务器的保护模式。
`initSentinel` 函数负责将 command table 设置为只支持 sentinal 命令，以及初始化 `sentinalState` 数据格式。

# 修复持久化文件

启动模式如果是 `redis-check-rdb/aof`, 那么就会执行 `redis_check_rdb_main` 或 `redis_check_aof_main` 这两个函数来修复持久化文件，不过 `redis_check_rdb_main` 函数所做的事情在 `Redis` 启动过程中已经做了，所以这里不需要做，直接使用这个函数就可以。

# 处理参数

如果是简单的参数例如 `-v` 或 `--version`, `-h` 或 `--help`, 就会直接调用响应的方法，打印信息。如果是使用其他配置文件，则修改 `server.exec_argv`. 对于其他信息，会将其转换为字符串，然后添加进配置文件，例如 `--port 6380` 就会转换成 `port 6380\n` 加进配置文件。此时，redis 就会调用 `loadServerConfig` 函数来加载配置文件，这个过程会覆盖前面初始化默认配置文件的变量的值。

# initServer

initServer 函数负责结束 server 变量初始化工作。首先设置处理信号(`SIGHUP` 和 `SIGPIPE` 除外), 接着会创建一些双向列表用来跟踪客户端，从节点等.

```cpp
server.current_client = NULL;
server.clients = listCreate();
server.client_index = raxNew();
server.client_to_close = listCreate();
server.slaves = listCreate();
server.monitors = listCreate();
server.client_pending_write = listCreate();
server.slaveseldb = -1;
server.unblocked_clients = listCreate();
server.ready_keys = listCreate();
server.client_waiting_acks = listCreate();
```

# Shared object

`createSharedObject` 函数会创建一些 shared 对象保存在全局的 `shared` 变量中，对于不同的命令，可能会有相同的返回值。这样返回时就不需要新增对象，保存到内存中。
这样设计以启动 Redis 时多消耗一些时间为代价，换取运行的更小的延迟

```golang
shared.crlf = createObject(OBJ_STRING, sdsnew("\r\n"));
shared.ok = createObject(OBJ_STRING, sdsnew("+OK\r\n"));
shared.err = createObject(OBJ_STRING, sdsnew("-ERR\r\n"));
shared.emptybulk = createObject(OBJ_STRING, sdsnew("$0\r\n\r\n"));
shared.czero = createObject(OBJ_STRING, sdsnew(":0\r\n"));
```

# shared integers

除了上述的一些返回值以外，`createSharedObject` 函数还会创建一些共享的整数对象。对 Redis 来说，有许多类型 (比如 lists 或者 sets) 都需要一些整数 (比如数量), 这是就可以复用这些已经创建好的整数对象，而不需要重新分配内存并创建。这同样是牺牲了启动时间来换取运行时间。

# 新增循环事件

`initServer` 函数调用 `aeCreateEventLoop` 函数 (`ae.c` 文件)来增加循环事件，并将结果返回给 server 的 el 成员。Redis 使用不同的函数来兼容各个平台，在 Linux 平台使用 `epoll`, 在 BSD 使用 `kqueue`, 都不是的话，最终会使用 `select`. Redis 轮询新的连接以及 I/O 事件，有新的事件到来时就会及时作出响应。

# 分配数据库

Redis 初始化需要的数据库，并将结果赋予 server 的 db 成员。

```cpp
server.db = zmalloc(sizeof(redisDb) * server.dbnum);
```

# 监听 TCP 端口

`listenToPort` 用来初始化一些文件描述符，从而监听 server 配置的地址和端口。`listenToPort` 函数会根据参数中的地址判断要监听的是 IPv4 还是 IPv6, 对应的调用 `anetTcpServer` 或者 `anetTcp6Server` 函数，如果参数中未指明地址，最会强行绑定 `0.0.0.0`

# 初始化 LRU 键池

`evictionPoolAlloc` (evict.c 文件中) 用于初始化 LRU 的键池，Redis 的 key 过期策略是近似 LRU 算法.

```cpp
void evictionPoolAlloc() {
    struct evictionPoolEntry *ep;
    int j;

    ep = zmalloc(sizeof(*ep)*EVPOOL_SIZE);
    for (j = 0; j < EVPOOL_SIZE; j++) {
        ep[j].idle = 0;
        ep[j].key = NULL;
        ep[j].cached = sdsnewlen(NULL, EVPOOL_CACHED_SDS_SIZE);
        ep[j].dbid = 0;
    }
    EvictionPoolLRU = rp;
}
```

# server cron

`initServer` 函数接下会为数据库和 pub/sub 再生成一些列表和字段，重置一些状态，标记系统启动时间。在这之后，Redis 会执行 `aeCreateTimeEvent` (在 `ae.c` 文件中) 函数，用来新建一个循环执行 `serverCron` 函数的事件。`serverCron` 默认每 100 毫秒执行一次.

```cpp
if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
    serverPanic("Can't create event loop timers.");
    exit(1);
}
```

可以看到，代码中创建循环事件时指定每毫秒执行一次 `serverCron` 函数，这是为了使循环马上启动，但是 `serverCron` 函数的返回值又会被作为下次执行的时间间隔。默认为 1000/server.hz.
`server.hz` 随着客户端数量的增加而增加。

`serverCron` 函数做了许多定时执行的任务，包括 `rehash`, 后台持久化，AOF 重新与清理，清理过期 key, 交换虚拟内存，同步主从节点等等。总之能想到的 Redis 的定时任务几乎都在 `serverCron` 函数中处理。

# 打开 AOF 文件

```cpp
if (server.aof_state == AOF_ON) {
    server.aof_fd = open(server.aof_filename,
                         O_WRONLY | O_APPEND| O_CREAT, 0644);
    if (server.aof_fd == -1) {
        serverLog(LL_WARNING, "Can't open the append-only file:%s",
                  strerror(errno));
        exit(1);
    }
}
```

# 最大内存限制

对于 32 位系统，最大内存是 4GB, 如果用户没有明确指出 Redis 可使用的最大内存，那么这里默认限制为 3GB.

```cpp
if (server.arch_bits == 32 && server.maxmemory == 0) {
    serverLog(LL_WARNING, "WARNING: 32 bit instance detected but no memory set. Setting 3 GB maxmemory limit with 'noeviction' policy now.");
    server.maxmemory = 3072L * (1024 * 1024);
    server.maxmemory_policy = MAXMEMORY_NO_EVICTION;
}
```

# Redis Server 启动

如果 Redis 被设置为后台运行，此时 Redis 会尝试写 pid 文件，默认路径是 `/var/run/redis.pid`. 这时，Redis 服务器已经启动。

# 从磁盘加载数据

如果存在 AOF 文件或者 dump 文件 (都有的话 AOF 文件的优先级高), `loadDataFromDisk` 函数负责将数据从磁盘加载到内存

# 最后的设置

每次进入循环事件时，要调用 `beforeSleep` 函数，主要做如下工作

- 如果 server 时 cluster 中的一个节点，调用 `clusterBeforeSleep` 函数
- 执行一个快速的周期
- 如果有客户端在前一个循环事件被阻塞了，向所有的从节点发送 ACK 请求
- 取消在同步备份过程中被阻塞的客户端的阻塞状态
- 检查是否有因为阻塞命令而被阻塞的客户端，如果有，解除
- 把 AOF 缓冲区写入磁盘
- 线程释放 GIL

# 进入主循环事件

程序调用 `aemain` 函数，进入主循环，这是其他的一些循环事件也会分别被调用

```cpp
void aemain(aeEventLoop *eventLoop) {
    eventLoop -> stop = 0;
    while (!eventLoop -> stop) {
        if (eventLoop -> beforesleep != NULL)
            eventLoop -> beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS | AE_CALL_AFTER_SLEEP);
    }
}
```

执行到这里，Redis server 已经准备好处理各种时间了。