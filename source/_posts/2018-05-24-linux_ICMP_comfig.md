---
title: Linux 下使用 ICMP 套接字
date: 2018-05-24
category: socket
---

　　在 Linux 下使用如下方法创建 ICMP 套接字

```C++
int sock = socket(PF_INET, SOCK_DGRAM, IPPROTO_ICMP);
```

　　发现即使切换到 root 用户仍然会提示 `permission denied`, 搜索 stackoverflow 有人建议执行如下命令
```shell
sudo sysctl -w net.ipv4.ping_group_range="0 0"
```
　　这行命令定义了可以创建 `non-raw ICMP socket`(非原始 ICMP 套接字~~这么翻译?~~) 的用户组的范围，默认是 `1 0`, 任何人都不可以，即使是 root 用户.
　　实际上这行命令更改了 `/proc/sys/net/ipv4/ping_group_range` 文件的内容，关于 `/proc` 目录，它的挂载点并不是磁盘文件系统，它是一个伪文件系统，只存在于内存中，允许在运行时访问内核和进程的信息.