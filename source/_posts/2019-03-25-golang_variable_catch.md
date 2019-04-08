---
title: golang 踩坑记(1)
category: Golang
date: 2019-03-25 18:00
tags: [bug]
description: 关于 golang 中的协程调度和变量捕捉
---

# 问题

在写代码时会有如下场景，有一个任务序列，用 for 循环分发，由于各个任务之间并不关联，希望用 go 并发执行各个任务，大致代码如下

```golang
for i := start; i < end; i += step {
    go func() {
        // figure task with i
    }()
}
```

其中用循环变量 i 用来指定任务区间，多次发现 goroutine 中的 i 与实际预期不符，尤其是 `end` 多次出现

# 原因

意识到由于 goroutine 调度和变量捕捉导致的。
由于在匿名函数中捕捉外部变量，所以当 routine 到执行态时捕捉到的 i 值并不是当时 goroutine 创建时的值，而是当前的值，因为并发后并不会立刻执行，所以当循环结束后捕捉到 i == end, 所以会出现 end 任务重复的问题。

# 解决方式

goroutine 中的任务标识不再从外部捕捉，而是改为函数调用，go 指定的并发函数的参数是在当前语句中确定的。
代码如下

```golang
for i := start; i < end; i += step {
    go func(i int) {
        // Do task i
    }(i)
}
```

# tips

不能假设并发程序中代码的执行顺序