---
title: golang 中 map 的并发安全
category: golang
date: 2019-09-09 17:00
description: golang 踩的一个并发安全的坑
---

原来以为 golang 这种原生支持并发的语言，内置类型都是并发安全的，结果这次出现了一个 bug, 排查了半天发现是 golang 中的 map 原来是不支持并发的，并发的读和写操作会导致进程 panic.

# golang 官网对 map 的并发安全的解释

> Maps are not safe for concurrent use: it's not defined what happens when you read and write to them simultaneously. If you need to read from and write to a map from concurrently executing goroutines, the accesses must be mediated by some kind of synchronization mechanism. One common way to protect maps is with sync.RWMutex.

map 在 golang 中并不是并发安全的结构，所以在多个 goroutine 中进行读写的场合，需要我们使用锁结构来保证读写的先后顺序，建议使用 `sync.RWMutex` 读写锁来进行约束。

> After long discussion it was decided that the typical use of maps did not require safe access from multiple goroutines, and in those cases where it did, the map was probably part of some larger data structure or computation that was already synchronized. Therefore requiring that all map operations grab a mutex would slow down most programs and add safety to few. This was not an easy decision, however, since it means uncontrolled map access can crash the program.
> The language does not preclude atomic map updates. When required, such as when hosting an untrusted program, the implementation could interlock map access.
> Map access is unsafe only when updates are occurring. As long as all goroutines are only reading—looking up elements in the map, including iterating through it using a for range loop—and not changing the map by assigning to elements or doing deletions, it is safe for them to access the map concurrently without synchronization.
> As an aid to correct map use, some implementations of the language contain a special check that automatically reports at run time when a map is modified unsafely by concurrent execution.

以上是对 golang 中 map 为什么不是并发安全的解释，如果 map 是原生支持并发的话，那么就需要在内部使用额外的锁结构，这无疑会拖慢大部分不需要锁的应用的性能。

map 只有当在写操作没有加锁时是不安全的，即多个并发的读取时是不会出现问题的，但是当有读写的同时进行，或者多个写的同时进行，就会产生 `map concurrent read and write` 异常.
