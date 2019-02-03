---
title: bazel 初探 
date: 2018-09-26 11:00
description:  google 开源的自动构建系统
tags: [bazel]
---

bazel 是 google 开源的自动构建系统。类似于 CMake, Make, Maven. bazel 支持项目使用多种语言编写，构建到多平台上。官方网址见链接见[链接](https://bazel.build/)

目前的需求主要是构建 C++ 代码文件。主要学习下 bazel 构建 C++ 的方法。

# bazel 工作流程

- **载入** 与 target 关联的 `BUILD` 文件
- **分析** 输入以及它们的依赖，应用特殊构建规则，产生动作图
- **执行** 构建动作直到生成最终构建目标

# Bazel build

## workspace

在构建项目之前，需要设置 workspace。 workspace 是包含项目源代码和 bazel 构建输出的目录。
包含如下被 bazel 专用的特殊文件

- `WORKSPACE`
  bazel 将包含该文件的目录及其子目录和文件识别为 bazel workspace.该文件位于工程的根目录
- `BUILD`
  一个或者多个 `BUILD` 文件。bazel 通过该文件构建工程的不同部分。(在 workspace 下的目录，如果包含 `BUILD` 文件则被视为一个 package)

为指定一个文件夹是 bazel 的 workspace, 可以在目录中创建一个空的 `WORKSPACE` 文件。
当 bazel 构建工程时，所有的输入和依赖都必须位于相同的 workspace.

## `BUILD`

`BUILD` 文件包含若干不同种类的 bazel 指令。其中最重要的指令是 build 指令，告诉 bazel 如何构建输出文件，例如可执行文件或者链接库文件。
每个在 `BUILD` 中的 build 规则都被称作一个 target, 并且指向一系列的源文件和依赖。一个 target 同样可以指向另一个 target.

一个简单的 `BUILD` 文件示例

```bazel
cc_binary(
    name = "hello-world",
    src = ["hello-world.cc"]
)
```

`cc_binary` 是 bazel 的内建规则。作用是指示 bazel 从 `hello-world.cc` 构建一个不带依赖自包含的可执行文件。

target 中的 attributes 确定该 target 的依赖和选项。 `name` 为必须属性，其他许多为可选属性。在示例中， `hello-world` 中的 `name` 属性是自解释的，`srcs` 确定 bazel 构建该 target 所需的源文件。

## 构建工程

在包含 `WORKSPACE` 的目录执行一下命令构建工程, 以 example 中的 `cpp-tutorial/stage1` 为例

```shell
bazel build //main:hello-world
```

bazel 将构建输出放在 `workspace` 下的 `bazel-bin` 目录中。

## 依赖图

可以使用以下命令生成工程的依赖图的文字形式，输出内容可以在[链接](http://www.webgraphviz.com/)中生成可视图。

```shell
bazel query --nohost_deps --noimplicit_deps 'deps(//main:hello-world)' --output graph
```

# 改进 bazel 文件

当项目较大时，可以将其分割为几个 target 和 package 允许快速增量编译(只重新构建更改过的文件)。
以 `cpp-tutorial/stage2` 为例

```bazel
cc_library(
  name = "hello-greet",
  src = ["hello-greet.cc"],
  hdrs = ["hello-greet.h"]
)

cc_binary(
  name = "hello-world",
  srcs = ["hello-wrold.cc"],
  deps = [
      ":hello-greet",
  ]
)
```

`cc_library` 规则构建 `library` 文件。 `hello-world`  target 中的 `deps` 表示 `hello-world` 依赖于 `hello-greet`.



~~这样效率好低，好乱~~