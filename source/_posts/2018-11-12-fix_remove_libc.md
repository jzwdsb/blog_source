---
title: 误删 libc.so.6 的解决办法
cateogry: Linux
date: 2018-11-12
---

之前有个任务要更新集群中的 `gnulibc`, 当安装上新的库把原来的 symbol link `libc.so.6` 删掉后所有的命令都不能使用，提示缺少 `libc.so.6`,　/(ㄒｏㄒ)/~~.
这个库提供的功能过于基础，基本 `linux` 下的所有的可执行文件都要运行时链接，如果要修复的话会用到 `ln` 命令，但是 `ln` 也依赖于 `libc.so.6`, 这就是个循环问题了。

运行时需要指定 `LD_PRELOAD`, `LD_PRELOAD` 是用于指定动态库的加载，优先级高于 `LD_LIBRARY_PATH`, 这样执行 `ln` 命令时显式指定要载入的 `share object`.

```shell
LD_PRELOAD=/path/to/libc-2.23.so ln -s /path/to/libc-2.23.so /path/to/libc.so.6
```

总结，类似 `gnulibc` 这种提供基础功能的库，最好只通过系统的包管理工具升级。