---
title: golang 的性能分析
category: Golang
date: 2019-01-15 20:00
---

golang 的性能分析分为以下三个方面

- profile
- GC&GCDEBUG
- Trace

# Profile

golang 的 profile 分为以下两个方面

- CPU 分析
  在 runtime 中，每隔很短的时间，记录当前正在运行的协程的栈。持续一段时间，通常是 5-10s。通过分析这段时间记录下来的栈，出现频率比较高的函数则占用 CPU 比较多

- Mem 分析
  只能分析在堆上申请内存的情况，同 CPU 类似，也是采用采样的方法，每一定此时的内存申请操作会采样一次。通过分析这些采样的记录可以判断哪些语句申请内存较多。

以上 profile 的原理均是基于采样。