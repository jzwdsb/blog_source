---
layout: post
category: Linux
date: 2018-04-03
title: linux I/O 多路复用模型
description: select, poll, epoll 接口
---

　　I/O 多路复用是指内核一旦发现进程指定的一个或多个 I/O 条件已经准备好可以操作，就通知该进程．<br>
　　由于在 linux 上一切皆为文件，所以一个 I/O 请求一定对应于一个文件描述符，这个文件描述符可以是一个 socket,device 或者一个普通文件.<br>
　　I/O 多路复用可以用作并发事件驱动程序的基础，在事件驱动程序中，某些事件会导致流向前推进．一般的思路是将逻辑流化为状态机.

　　I/O 多路复用适用于以下场合

1. 当客户处理多个描述符时(一般是交互式输入和网络套接口)，必须使用 I/O 复用
2. 当一个客户同时处理多个套接字时.
3. 如果一个 TCP 服务器既要处理监听套接字，又要处理已连接套接字，一般也要用到 I/O 复用
4. 如果一个服务器既要处理 TCP．又要处理 UDP，一般要使用 I/O 复用
5. 如果一个服务器要处理多个服务或多个协议，一般要使用 I/O 复用

　　与多进程与多线程计数相比，I/O 多路复用技术的最大优势是系统开销少，系统不必创建进程线程，目前支持 I/O 多路复用的系统调用有 `select`, `pselect`, `poll`, `epoll`．

## select

### 函数原型

```C
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeouw);
```

### 接口描述

　　监视并等待多个文件描述符的属性变化(可读，可写或者错误异常). `select` 函数监视的文件描述符分为 3 类，分别是 `writefds`, `readfds` 和 `exception`. 调用后 `select` 后阻塞,直到有描述符就绪(有数据可读,可写,或者有错误异常)，或者超时( timeout 指定时间)，函数才返回．当 `select` 返回后，可以通过遍历 fdset，来找到就绪的描述符.

![select_IO](/downloads/select_IO.jpg)

### 参数

* nfds:

    要监视的文件描述符的范围，一般取监视的描述符数的最大值加 1, 如这里写 10, 这样的话，描述符 0, 1, 2 ... 9 都会监视．在 linux 上这个参数的最大值默认为 1024, 这个值由 FD_SIZE 宏定义，可以修改这个值并且重新编译内核更改 ndfs 的取值范围.
* readfd

    监视的可读描述符集合
* writefds

    监视的可写描述符集合
* exceptfds

    监视的错误描述符集合
* timeout

    超时时间，这个参数有三种可能
    1. NULL: 永远等待，仅在一个描述符准备好 I/O 时才返回
    2. 等待固定时间
    3. 不等待，timeout 变量时间设置为 0 秒

　　`select` 的缺陷就是单个进程打开的 FD 是有一定限制的，它由 FD_SIZE 设置，默认为 1024．<br>
　　对 socket 扫描时是线性扫描，即采用轮询的方法，效率低.<br>
　　需要维护一个用来存放大量 fd 的数据结构,这样会使用户空间和内核空间在传递该结构时复制开销大.


## poll

### 函数原型

```C
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

### 接口描述

　　`poll` 本质上和 `select` 没有区别，他将用户传入的数组拷贝到内核空间，然后轮询每个 fd 对应的设备状态,如果设备就绪则在设备等待队列中加入一项并继续遍历,如果遍历完所有 fd 后没有发现就绪设备,则挂起当前进程,直到设备就绪或者主动超时,被唤醒后它又要再次遍历 fd，这个过程经历了多次无谓的遍历.<br>
　　它没有最大连接数的限制，原因在于它是基于链表保存的.


### 参数

* fds

    `poll` 使用一个 pollfd 的指针表示描述符集合.
* nfds

    用来指定第一个参数数组的元素个数
* timeout

    指定等待的毫秒数

　　`poll` 的缺陷在于，大量的 fd 的数组被整体复制于用户态和内核地址空间之间，不管这样的复制是不是有意义.<br>
　　`poll` 的另一个特点是水平出发，即如果报告了 fd 后，没有被处理，那么下次 poll 时会再次报告 fd.

## 注意

　　从上面看，`select` 和 `poll` 都需要在返回后，通过遍历文件描述符来获取已就绪的 socket. 事实上，同时连接的大量客户端在同一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降

## epoll

### 接口描述

　　`epoll` 是在 2.6 内核中提出的,是之前 `select` 和 `poll` 的增强版本．相对于 `select` 和 `poll` 来说，`epoll` 更加灵活，没有描述符限制．它使用一个文件描述符管理多个描述符，将用户关系的文件描述符的时间存放到内核的一个时间表中，这样在用户空间和内核空间的 copy 只需一次．<br>
　　`epoll` 支持水平触发和边缘触发，最大的特点在于边缘触发，它只告诉进程那些 fd 刚刚变为就绪态，并且只会通知一次．还有一个特点是，`epoll` 使用**事件**的就绪通知方式，通过 `epoll_ctl` 注册 fd, 一旦该 fd 就绪，内核就会采用类似 callback 的回调机制来激活 fd, `epoll_wait`便可收到通知.<br>
　　epoll 操作过程需要三个接口，分别如下

```C
#include <sys/epoll.h>

int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

### epoll_create

　　该函数生成一个 epoll 专用的文件描述符，(创建一个 epoll 的句柄)，可见于 `/proc/<process-id>/fd` 下．<br>
　　size 在 linux 2.6.8 之后，size 参数是被忽略的.<br>
　　成功返回 `epoll` 专用的文件描述符，失败返回 -1

### epoll_ctl

　　`epoll` 的事件注册函数，它注册要监听的事件类型

* epffd: epoll 专用的文件描述符，epoll_create 的返回值
* op: 表示动作
* fd: 需要监听的文件描述符
* event: 告诉内核要监听什么事件

### epoll_wait

　　等待事件的产生，收集在 `epoll` 监控的事件中已经发送的时间，类似于 select 调用

* epfd: epoll 专用的文件描述符
* eventsL 分配好的 epoll_event 结构体数组
* maxevents: maxevents 告知内核这个 events 有多大
* timeout: 超时时间，单位为毫秒

　　`epoll` 的没有最大并发连接的限制，能打开的 FD 的上限远大于 1024(1G 的内存上能监听约 10 万个端口)．<br>
　　效率提升，不是轮询的方式，不会随着 FD 数目的增加效率下降．只有活跃可用的 FD 才会调用 callback 函数，因此在实际的网络环境中， epoll 的效率就会远高于 select 和 poll.<br>
　　内存拷贝，利用 mmap 文件映射内存加速与内核空间的消息传递，即 epoll 使用 mmap 减少拷贝开销

　　综上，在选择 `select`, `poll`, `epoll`时要根据具体的使用场合以及这三种方式的自身特点
1. 表面上看 `epoll` 的性能最好，但是在连接数少，并且连接都十分活跃的情况下， `select` 和 `poll` 的性能可能比 `epoll` 好，毕竟 epoll 的通知机制需要很多函数回调
2. `select` 低效是因为它都需要轮询，但低效也是相对的，视情况而定，也可通过良好的设计改善.
