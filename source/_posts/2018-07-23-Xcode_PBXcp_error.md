---
title: Xcode PBXcp 错误及其解决办法
category: Mac
tags: [bug, Xcode]
date: 2018-07-24
---

删除 `~/Library/Developer/Xcode/DerivedData` 中的所有内容

```shell
rm -rf ~/Library/Developer/Xcode/DerivedData/*
```

重启 `Xcode`。

写了两天的 bug, 心累