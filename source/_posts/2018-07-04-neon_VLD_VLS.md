---
title: ARM Neon 编程(二) 访存指令详解
description: VLD, VST
category: ARM
tags: [ARM, Neon]
date: 2018-07-04
---

## VLDn

根据 list 参数的不同可以将 VLDn 分为一下三种

- single n - element to one lang
    读取一个 n - element 结构体到寄存器的单个通道中
- single n - element to all lang
    读取 n - element 结构体到寄存器的所有通道中
- multiple n - element structures
    读取 n - element 结构体到寄存器。n 指定分选模式

### 语法格式

```asm
VLDn{cond}.datatype list,  [Rn{@aligin}]{!}
```

- n
    必须在 {1, 2, 3, 4}
- cond
    可选的状态码
- datatype
    单个元素的 bits 数，通常为 8, 16, 32
- list
    包括在 {} 中的高级 SIMD 寄存器列表
- Rn
    通用寄存器，包含基地址，Rn 不能为 PC
- align
    指定可选的对齐，通常为 16 的倍数。
- !
    如果存在，那么 Rn 寄存器将在操作后更新(Rn + 指令读取的字节数)
- Rm
    通用寄存器，包含从基地址的偏移量，如果 Rm 存在，当指令访问内存结束后，Rn 更新(Rn=Rn + Rm), Rm 不能为 PC 或 SP。

### list 参数

- 读取一个结构，单通道
  - {Dd[x], D(d+1){x}}
  - {Dd[x], D(d+2){x}, D(d+4){x}}
- 读取一个结构，全通道
  - {Dd[], D(d+1)[]}
  - {Dd[], D(d+2)[],D(d+4)[]}
- 读取多个结构，支持通道分选
  由 n, datatype 共同决定通道分选模式
  - {Dd, D(d+1)}
  - {Dd, D(d+2)}

## VLDM

### 语法格式

```asm
VLDMmode{cond} Rn{!} Registers
```

- mode
  - IA
    表示每次传输完成后地址向高地址增长。默认模式并且可以忽略
  - DB
    表示每次传输完成后地址向低地址增长。
  - EA
    Empty Ascending Operation.(~~这什么意思~~), 在传输时与 DB 相同。
  - FD
    Full Descending Stack Operation. 在传输时与 IA 相同。
- cond
  可选的条件码。
- Rn
  保存传输时基地址的通用寄存器
- !
  可选。表示基地址更新后必须写会 Rn, 如果没有 ! 选项，那么模式必须是 IA 模式
- Registers
  在 {} 中的寄存器列表。D 寄存器和 Q 寄存器不能混用。不能超过 16 个 D 寄存器或者 8 个 Q 寄存器。

## VSTn

### 语法格式

```asm
VSTn{cond}.datatype list [Rn{@align}] {!}
VSTn{cond}.datatype list [Rn{@align}]，Rm
```

- n
  必须在 {1, 2, 3, 4} 中
- cond
  可选的条件码
- datatype
  8, 16, 32, 64
- list
  在 {} 中的高级 SIMD 寄存器列表。
- Rn
  保存基地址的通用寄存器，不能是 PC
- align
  可选，表示指针对齐
- ！
  可选，Rn 将在存储生效后更新。(Rn + 指令传输的内存数)
- Rm
  通用寄存器，保存从基地址开始的偏移量。Rn 将在访存生效后更新(Rn = Rn + Rm). Rm 不能使 SP 或者 PC

list 参数的不同格式对 VST 效果同 VLD

## VSTR

扩展寄存器存储(extension register store)

```asm
VSTR{cond}{.64} Dd, [Rn [, #offset]]
```

- cond
  可选的条件码
- Dd
  要保存的扩展寄存器
- Rn
  保存访存目的地址的通用寄存器
- offset
  可选的数学表达式。满足以下条件
  - 必须在 assembly 时期就可以求值
  - 必须是 4 的倍数
  - 在 [-1020, 1020] 中间
  在传输时基地址会加上该偏移量

VSTR 指令用于向内存保存扩展寄存器的内容
一次传输两字。