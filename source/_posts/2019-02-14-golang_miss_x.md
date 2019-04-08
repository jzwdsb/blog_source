---
title: golang 缺失 sys 问题
category: Golang
date: 2019-02-14
---

golang 缺失 golang/x/sys 包，可是这个属于核心组件，应该是不会缺失的，只能手动下载了

进入 `$GOPATH/src/golang.org/x/`, 手动 clone 相关代码

```shell
cd $GOPATH/src/golang.org/x/
git clone https://github.com/golang/sys.git
```

问题解决，类似问题可以通过相同方法
