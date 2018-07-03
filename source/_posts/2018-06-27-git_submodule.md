---
title : git 下载子模块
category: git
date: 2018-06-27
---

下载的 repository 依赖于子模块，但是 clone 这个 repository 时子模块不会被 clone 下来，要完整编译运行这个 project 需要把其子模块下载下来.

切换 `pwd` 到这个 repository 执行如下命令，下载其依赖的全部子模块

```shell
git submodule update --init --recursive
```