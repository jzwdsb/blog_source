---
title: Cmake 生成 Xcode 工程
date: 2018-07-23
tags: [Cmake, Mac, Xcode]
---

在 Mac OSx 上从包含 CmakeLists.txt 的文件夹生成 Xcode 工程

```shell
cmake -G Xcode -H . -B build
```

这行命令从当前目录读取 `CMakeLists.txt` 文件并创建 `build` 目录并在其下生成 `Xcode` 工程。