---
date: 2018-05-01
category: Linux
title: Linux 下 socket 编程 API 汇总（一）
description: 必要的工具
---

## 套接字地址结构

### IPv4 套接字地址结构

```C
struct in_addr{
    in_addr_t   s_addr;
};

struct sockaddr_in{
    uint8_t         sin_len;
    sa_family_t     sin_family;
    in_port_t       sin_port;
    struct in_addr  sin_addr;
    char            sin_zero[8];     
};
```

　　POSIX 规范只需要这个结构中的三个字段, `sin_family`, `sin_addr` 和 `sin_port`, 其中，`sin_family` 字段表示地址族，对于 IPv4 套接字来说，恒为 `AF_INET`, `sin_addr` 字段表示网络字节序下 32 位的 IPv4 地址．`sin_port` 字段表示 16 位的 TCP 或者 UDP 端口号.

### 通用套接字地址结构

```C
struct sockaddr
{
    uint8_t         sa_len;
    sa_family       sa_family;
    char            sa_data[14];
};
```

　　不同协议族的套接字地址结构类型参数不同，但是所有套接字函数都应有相同的接口，为了解决这个问题，在 `<sys/socket.h>` 中定义一个通用的套接字地址结构，于是套接字函数的地址结构参数定义为一个指向通用地址结构的指针.以 `bind` 为例

```C
int bind(int sockfd, struct sockaddr* addr, socklen_t addrlen)
```

### IPv6 套接字地址结构

```C
struct in6_addr
{
    uint8_t         s6_addr[16];
};

#define SIN6_LEN

strcut sockaddr_in6
{
    uint8_t             sin6_len;
    sa_family           sin6_family;
    in_port_t           sin6_port;
    uint32_t            sin6_flowinfo;
    struct  in6_addr    sin6_addr;
    uint32_t            sin6_scope_id;
};
```

　　对于上述地址结构要注意以下结点
- 如果系统支持套接字地址结构中的长度字段，那么 `SIN6_LEN` 常值必须定义
- IPv6 的地址族是 `AF_INET6`, IPv4 的地址族为 `AF_INET`
- 结构中字段的先后顺序做过编排，使得如果 `sockaddr_in6` 结构本身是 64 位对齐的，那么 128 位的 `sin6_addr` 字段也是 64 位对齐的．在一些 64 位处理机上，如果 64 位数据存储在某个 64 位边界位置，那么对它的访问将得到优化处理
- `sin6_flowinfo` 字段分为两个字段
    - 低序 20 位是流标(flow label)
    - 高序 12 位保留
- 对于具备范围的地址(scoped address), `sin6_scope_id` 字段标识其范围(scope), 最常见的是链路局部地址(link-local address)的接口索引(interface index)

## 值-结果参数

　　当向一个套接字函数传递一个套接字地址结构时，该结构总是以引用(也就是指针)形式传递．该结构的长度也作为一个参数传递，但其传递方式取决于该结构的传递方向: 从进程到内核，还是从内核到进程

- 从进程到内核
    - bind
    - connect
    - sendto

　　参数包括套接字地址结构指针和和该结构的整数大小

- 从内核到进程
    - accept
    - recvfrom
    - getsockname
    - getpeernam

　　参数包括一个套接字地址结构指针和指向表示该结构大小的整数变量的指针．
　　当该函数被调用时，结构大小是一个值，告诉内核该结构的大小以至于写该结构时不至于越界.
　　当函数返回时，结构大小是结果，它告诉进程内核在该结构中究竟存储了多少信息.
　　这种类型的参数被称为 值-结果(value-result)参数

　　在网络编程中，值-结果参数最常见的例子是所返回套接字地址结构的长度
- `select` 函数中间的三个参数
- `getsockopt` 函数的长度参数
- `recvmsg` 函数，`msghdr` 结构中的 `msg_namelen` 和 `msg_controllen` 字段
- `ifconf` 结构中的 `ifc_len` 字段
- `sysctl` 函数两个长度参数中的第一个

## 字节排序函数

　　由于主机使用的字节序和网际协议使用的字节序可能不同，所以使用字节排序函数实现主机字节序到网络字节序的转换，对于主机字节序与网络字节序相同的主机，字节排序通常定义为空宏.
　　由于历史和 POSIX 规范的约定，套接字地址结构中的某些字段必须按照网络字节序进行维护.
　　在主机与网络字节序之间转换使用以下四个函数
```C
#include <netinet/in.h>

uint16_t htons(uint16_t host16bitvalue);

uint32_t htonl(uint32_t host32bitvalue);

uint16_t ntohs(uint16_t net16bitvalue);

uint32_t ntohl(uint32_t net32bitvalue);
```

　　这些接口名可以这样理解， **h**表示 **host**, **n** 表示 **network**, **s** 表示 **short**, 代表 16 位长的无符号数, **l** 表示 **long**, 代表 32 位长的无符号整数

## 字节操纵函数

　　操纵多字节字段的函数有两组，分别起源于 4.2BSD 和 ANSI C, 他们既不对数据做解释，也不假设数据是以空字符结束的 C 字符串．
　　名字以 b(byte) 开头的第一组函数起源于 4.2 BSD, 名字以 mem(memory) 开头的第二组函数起源于 ANSI C 标准. 
###　4.2 BSD

```C
#include <string.h>

/** 将目标字节串中指定数目置为０*/
void bzero(void* dest, size_t nbytes);
/** 将指定数目的字节从源字节串拷贝到目标字节串*/
void bcopy(const void* src, void* dst, size_t nbytes);
/** 比较两个字符串，相同返回 0, 不同返回最后不同的字节的差*/
int bcmp(const void* ptr1, const void* ptr2, size_t nbytes);
```

### ANSI C

```C
#include <string.h>

/** 将目标字节串指定数目的字节置为 c*/
void* memset(void* dst, int c, size_t len);
/** 从 src 中拷贝　nbytes 字节到 dst*/
void* memcpy(void* dst, const void* src, size_t nbytes);
/** 比较两个字节串，相等返回 0,  不等返回最后不等字节的差*/
void* memcmp(const void* ptr1, const void* ptr2, size_t nbytes);
```

## 地址转换函数

　　目前有两组地址转换函数，一组仅适用于 IPv4, 另一组对 IPv4 和 IPv6 都适用.

- 适用于 IPv4 
```C
#include <arpa/inet.h>

/** 若字符串有效则返回 1, 否则返回 0*/
int inet_aton(const char* strptr, struct in_addr* addrptr);
/** 若字符串有效则为 32 位二进制网络字节序的 IPv4 地址，否则为 INADDR_NONE*/
in_addr_t inet_addr(const char* strptr);
/** 将 IPv4 地址转换为点分十进制数串，返回该字符串的指针*/
char* inet_ntoa(struct in_addr inaddr);
```

- 通用地址转换函数
```C
#include <arpa/inet.h>

/** 若成功则返回 1, 若输入不是有效的表达格式则为 0, 若出错则为 -1*/
int inet_pton(int family, const char* strptr, void* addrptr);
/** 若成功则返回结果字符串指针，出错则返回 NULL*/
const char* inet_ntop(int family, const void* addrptr, char* strptr, size_t len);
```

