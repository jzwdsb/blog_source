---
title: golang 踩坑记(2)
date: 2019-04-07 16:00
category: Golang
description: golang 中 defer 的正确使用姿势
tags: [bug, 资源调度]
---

defer 在 golang 中用来实现延迟调用机制，defer 关键字确保其声明的被调用函数会在调用函数退出前正确调用，顺序沿 defer 声明顺序的逆序。

# 场景

在 for 循环中请求资源，并在该循环结束时使用 defer 将其关闭。

```golang
for i := start; i < end; i += step {
    // some resourse open with i
    // such as file
    defer close(file)
}
```

# 问题

这里的 defer 会在该函数结束时调用，申请的资源会在函数退出时集中释放，但是这里的资源声明周期是确定的，当一个循环结束时该资源的生命周期就该结束，如果集中在函数退出时释放，可能会出现该函数调用时占用资源不断增长至过大的问题，而改为循环结束时释放就不会出现该问题.

最佳实践如下

```golang
for i := start; i < end; i += step {
    //some resourse as file
    close(file)
}
```
