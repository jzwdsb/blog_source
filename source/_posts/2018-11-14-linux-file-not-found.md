---
title: Linux 下面执行可执行文件却报 file not found 错误
date: 2018-11-14 15:05:25
category: Linux
---

在一个 64-bit Arm 开发板上执行一个 32-bit 的程序，bash 却报 `file not found error`, 可是 `ls` 是可以看到这个文件存在。

`file not found error` 通常会有以下三种原因

- 文件确实不存在
- 该文件是一个悬挂的 symbolic link
- 这是个 32-bit 的 object(`file ${FILE} 查看该文件类型`), 但是 host 是 64-bit, 缺少 32-bit 的支持。

这次的错误就是第三个原因，运行该程序所依赖的运行时环境缺少关键组件，但是报告这个错误的通道只有空间使用这个错误码而并没有提供额外的信息。

可以查看 [stackexchange](https://unix.stackexchange.com/questions/13391/getting-not-found-message-when-running-a-32-bit-binary-on-a-64-bit-system/13409#13409) 得到更多信息。