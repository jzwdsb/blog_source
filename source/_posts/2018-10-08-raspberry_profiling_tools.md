---
title: Raspberry Pi Software profiling tools
category: Linux
tags: [Raspberry Pi, Software profiling]
date: 2018-10-08 20:00
---

Raspberry Pi 性能分析工具汇总

- gprof
- gperftools
- valgrind
- oproile

## gprof

gprof 属于 GCC 编译器工具包，安装 gcc 编译器后即可使用

### usage

- 编译选项加入 `-pg` 打开 gprof 分析模式
- 链接选项加入 `-all-static` 强制 linker 使用静态链接库
  gprof 分析只会覆盖可执行文件中的例程，运行时加载库中的函数不会在分析结果中，如果使用动态链接库，程序的启动方式会不同。


编译生成可执行文件后，像往常一样运行，分析信息和 benchmark 会输出到 `gmon.out` 文件中.

如果使用动态链接方式，使用 libtool 在运行文件时调用 gprof

```shell
libtool -mode=execute gprof ./demofile
```

当程序执行完毕后，使用 gprof 分析该程序的运行时的开销。

```shell
gporf ./demofile
```

### 注意

gprof 没有线程安全的实现，无法调试并行计算程序，如使用 `OpenMP` 或者 `pthread` 编写的程序。

## gperftools

gperftools 是 Google 提供的软件性能分析工具。原理基于时间的采样，能够正确分析多线程程序，所以可以分析使用 `OpenMP` 的程序。

### install

```
sudo apt install google-perftools libgoole-perftools-dev
```

### usage

- 在需要 profile 的代码段两端加上对 `ProfilerStart("demofile.prof")` 和 `ProfilerStop()` 的调用。接口声明在 `gperftools/profiler.h` 中
- 编译中加入 `profiler` 链接库, 强制使用静态链接, 动态链接库中的函数不会出现在 profiling results 中.
  `LDFLAGS+= "-all-static -lprofiler"`

在运行编译后的可执行文件后,使用 `goole-pprof` 生成运行时分析结果.

```shell
google-pprof -text ./demofile demo.prof
```

## valgrind

valgrin 是很复杂且使用较为广泛的性能与内存分析工具。它会将程序运行在一个虚拟的处理器中获得其运行时的全部信息。因为是运行在虚拟环境中，所以分析速度会比实时运行慢很多倍。
valgrind 对多线程程序如 `OpenMP` 的调试也不理想。

### install

```shell
sudo apt install valgrind kcachegrind
```

### usage

可见[官方文档](http://valgrind.org/docs/manual/manual.html)

## oprofile

oprofile 不在 Raspbian 官方 package 中，可以从源码编译获得。但是编译后立刻会报不适用于 Raspberry CPU 或 kernel.
至少需要内核级的重构或许才可以，但也不一定能运行于 Raspberry Pi 环境。

所以 `oprofile` 暂时不可用。
