---
title: golang 中的 select 关键字
category: golang
date: 2019-01-16 15:00
---

`select` 是 golang 中的一个控制结构，类似于 `switch`. 每一个 `case` 都必须为一个通信操作，要么是发送要么是接受。
`select` 随机选择一个可运行的 `case`, 如果没有 `case` 可以运行，便会阻塞，直到有 `case` 可以运行。一个默认的字句总是可以运行的。

```golang
select {
    case communication clause :
        statement(s)
    case communication clause :
        statement(s)
    default :
        statement(s)
}
```

以下描述 `select` 语句的语法

- 每个 `case` 都必须是一个通信
- 所有 `channel` 表达式都会被求值
- 所有被发送的表达式都会被求值
- 如果任意某个通信可以执行，它就会执行；其他就会被忽略
- 如果有多个 `case` 都可以运行，`select` 会随机公平的选出一个执行。其他不会执行。
  否则
  - 如果有 `default` 子句，则执行该语句
  - 如果没有 `default` 子句，`select` 将阻塞，直到某个通信可以执行；channel 或者值不会被重复求值

示例

```golang
package main

import "fmt"

func fibonacci(c, quit chan int) {
    x, y := 0, 1
    for {
        select {
        case c <- x:
            x, y = y, x+y
        case <-quit:
            fmt.Println("quit")
            return
        }
    }
}

func main() {
    c := make(chan int)
    quit := make(chan int)

    // start a goroutine to print current result
    // no buffer in c and quit channel, so this code
    // would block when this goroutine try to print
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(<-c)
        }
        quit <- 0
    }()
    fibonacci(c, quit)
}
```