---
title: golang 中的 goroutine 深入理解
category: golang
date: 2019-01-22 15:00
---

golang 内置对协程的支持，十分便于开发高并发程序，现在深入了解下 goroutine 的实现

# goroutine 的使用

使用 golang 的协程支持只需要使用 go 关键字即可，示例如下

```golang
// Create a new goroutine and execute the method
go DoSomeThing()
// Create a new anonymous method and run it in a goroutine
go func () {

}()
// Create a goroutine directly and execute the code block in the goroutine
go {

}
```

## 配置 runtime.GOMAXPROCS

可以使用 `runtime.GOMAXPROCS` 确定进程占用的核数

# 并行与并发的不同

## 进程，线程与处理器

现代 OS 中, 线程是 CPU 调度和执行的基本单位。进程是资源分配的基本单位。
每个进程包括私有虚拟地址空间，代码段，数据和其他的系统资源。
线程是一个进程中的一个执行单元。
一个进程至少会有一个线程。

## 并行与并发

对于并发和并行，应该从进程和线程的角度理解

### 并发

在一个时间段内，同时有线程在执行，但是在一个特定的时间点内，只有一个线程在执行。
多个线程通过调度算法比如(时间片)由 OS 调度执行

### 并行

在同一个时间点，有多个线程在执行。
并行程序需要硬件的支持，通常需要运行在多核处理器上

# 多个不同的多线程模型

## 用户态和内核态

多线程的实现可分为用户态线程和内核态线程

- 用户态线程
  由用户的代码实现
- 内核态线程
  由 OS 的支持

## 多线程模型

### 多对一模型模型

多个用户态的线程可对应于一个内核态的线程，线程的调度完全在用户空间实现。
用户态线程对 OS 不可见

由于多个用户态线程对应于一个内核态线程实例，如果该线程陷入内核态比如 I/O 请求，并且内核态线程在阻塞在 I/O 请求，所有的用户态线程都会阻塞，用户态线程可以使用非阻塞的 I/O 方法，但是同样会有性能和复杂性的问题

### 一对一模型

每个用户态线程都映射到内核态线程。

### 多对多模型

用户态线程和内核态线程的个数比为 M:N.
这样的模型需要内核态调度和用户态调度的交互，这样当发生线程的上下文切换时大部分发生在用户态，这样能够提高程序的性能，并且充分利用多核处理器的计算资源

# goroutine 机制的调度实现

goroutine 实现了上面的 M:N 模型，goroutine 机制的底层是使用了协程来实现。golang 内置的调度器能够允许协程在多核 CPU 上的执行。

## 调度器的工作方式

在 golang 内置的调度器中有四种结构，分别为 M, G, P 和 Sched
M, G, P 生命在 `runtime.h`, Sched 声明在 `proc.h` 中

### M 结构

表示系统内核态线程，并有 OS 管理。goroutine 在 M 上运行。
M 结构包含了很多底层相关信息，比如小对象缓存，当前正在执行的 goroutine, 随机数生成器

### P 结构

代表 CPU, 主要是为了执行 goroutine, 这个结构维护一个 goroutine 队列，保证 Sched 能够支持 N:1 到 M:N 不同模型的调度

### G 结构

是 goroutine 实现的核心结构。包括运行栈，指令指针 和其他调度 goroutine 的信息，比如阻塞的通道

### Sched 结构

调度器，维护一个存储 M 和 G 的队列以及其他调度器的状态

## 调度方式

在一个 M 结构中，会保存当前的上下文(P)，并且运行一个 goroutine 实例(G).
多个 goroutine 会绑定在对应的 P 中，同一时刻只有 goroutine 在运行，可是其他的 goroutine 会处于可调用或者阻塞的状态。

### 系统调用

当一个内核态线程陷入系统调用并且阻塞时，调度器会将绑定在这个 P 上的其他 goroutine 转移到其他 M 上。

### steal work

当在 context runqueue 不平衡时，调度器会将合适 M 上的处于等待状态 goroutine 实例转移到空闲的 M 上。

# 汇编层面的实现

示例代码

```golang
func test(i int) int {
    return i + 1
}

func demo() {
    go test(1)
}
```

重点关注 go 语句的汇编实现

```asm
pcdata  $2, $0
pcdata  $0, $0
movq    $1, 16(SP)
movl    $16, (SP)
pcdata  $2, $1
leaq    "".test·f(SB), AX
pcdata  $2, $0
movq    AX, 8(SP)
call    runtime.newproc(SB)
```

pcdata 指令用来包含一些垃圾收集器需要的信息，由编译器产生

主要逻辑为操作数入栈后调用 `runtime.newproc` 生成新的 goroutine。
