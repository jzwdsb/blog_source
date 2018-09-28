---
title: bazel 的 BUILD 文件的结构
date: 2018-09-26 15:00
category: bazel
---

通常 bazel 中的 `BUILD` 文件的结构如下

- package description(comment)
- 所有 `load()` 语句
- `package` 语句
- rules 和 marcos

## package description

包含对 package 的描述

## load 语句

bazel 扩展文件以 `.bzl` 结尾。使用 `load` 语句导入扩展文件中的符号

```bazel
load("//build_tools/rules:maprule.bzl", "maprule")
```

这行代码载入 `/build_tools/rules/maprule.bzl` 文件并且将 `maprule` 符号添加到环境中。
该命令通常用来载入 rules, functions 或者 constants(string, list etc.)

## package 语句

```bazel
package(default_deprecation, default_testonly, default_visibility, feature)
```

- default_visibility
  这个 package 中 rules 的可见度
- default_deprectation
  这个 package 的 rule 的默认 message
- default_testonly
  设置这个 package 所有 rules 的 testonly 属性的默认值
- feature
  设置不同的 flag, 影响当前的 BUILD 文件的语义

## rules

先记下见到的

- cc_library
- cc_binary