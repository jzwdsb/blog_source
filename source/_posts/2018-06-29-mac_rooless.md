---
title: Mac 关闭 rootless 模式
date: 2018-06-29
category: Mac
---

第一次使用Mac，发现在某些目录中间拷贝或者创建文件时即使使用 sudo 也提示 Operation Not permitted，经过搜索发现是 Mac 开启了 Rootless 模式，即使 root 用户也不能对一些文件夹如 `/System`, `/usr`, `/bin`, `/sbin` 做修改。 需要手动关闭这个模式。

重新启动，按住 Command + R 进入恢复模式

打开 Terminal

输入

```C++
csrutil disable
```

则关闭 Rootless 模式，重新打开 Rootless 模式，则输入

```C++
csrutil enable
```
