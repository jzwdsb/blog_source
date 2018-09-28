---
title: Linux 交叉编译 aarch64 平台
date: 2018-09-28 11:30
category: Linux
---

本地编译平台: Linux Ubuntu 16.04
目标编译平台: aarch64
工具链: bazel

编译 ops_test target, 相对路径为 `//mace/ops:ops_test`

## 简单步骤

- 本地使用 aarch64 编译器交叉编译到 aarch64 平台目标文件
- 将编译输出 push 到开发板上，运行测试

## 编译命令

bazel 指定 config 目标编译平台为 `aarch64_linux`

```shell
bazel build -s --config aarch64_linux --define openmp=true --define opencl=true --define neon=true //mace/ops:ops_test
```