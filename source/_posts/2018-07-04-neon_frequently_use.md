---
title: ARM Neon 常用指令
date: 2018-07-04 15:00
description: 记下编程中常用的指令
category: Neon
tags: [ARM, Neon]
mathjax: true
---

## 常用指令

看实现源码，常用 Neon 指令分为以下几类

- 访存指令
  - `pfrm`
  - `ld1`
  - `st1`
  - `pld`
  - `vld`
- 计算指令
  - `fmla`
  - `subs`
  - `add`
- 向量指令
  - `vmla`
  - `vand`
  - `ext`
  - `ins`
  - `trn1`
- 跳转指令
  - `bne`

## 访存指令

### `prfm`

内存预取指令(prefetch memory)

#### 语法格式

```asm
PRFM (prfop|#imm5), [Xn|SP{, #pimm}]
PRFM (prfop|#imm5), label
PRFM (prfop|#imm5), [Xn|SP, (Wm|Xm){, extend {amount}}]
```

- `prfop`
  预取操作方式。 具体格式为 `type` `<target>` `polciy`
  - `type`
    - `pld`
      数据载入预取
    - `PLI`
      指令载入预取
    - `PST`
      数据存储预取
  - `<target>`
    - `L1`
      L1 缓存
    - `L2`
      L2 缓存
    - `L3`
      L3 缓存
  - `policy`
    - `keep`
      保留或临时预取，在缓存中正常分配
    - `strm`
      流动或非临时预取，预取数据只使用一次
- `imm5`
  预取操作是否编码为立即数，在 [0, 31] 之间,只有在使用 `profp` 无法访问时才可以使用
- `Xn|SP`
  通用基址寄存器或者栈指针的名字
- `Wm`
  当 `option<0>` 设置为 0, 表示 32 bits 的寄存器
- `Xm`
  当 `option<0>` 设置为 1, 表示 64 bits 的寄存器。
- `pimm`
  可选的偏移量，整数。必须为 8 的倍数，在 [0, 32760] 之间。默认为 0。
- `label`
  要载入的数据的 `program label`. 是从本条指令开始的偏移量。在 $\pm$ 1MB 区间内
- `extend`
  extend/shift specifer 的索引。默认为 `LSL`, 当 `amount` 忽略时，此项默认为 `LSL`。可以是 `UXTW`, `LSL`, `SXTW`, `SXTX`.
- `amount`
  指数移位量(index shift amount),只有当 `extend` 不是 `LSL` 时可选。可以是 #0, #3.

#### 示例

```asm
prfm pldl1keep, [%1 #128]
```

### `ld1`

读取单个向量到寄存器

### `st1`

从 1 到 4 个寄存器向内存存储单个向量。

### `pld`

预取数据，允许处理器向内存管理系统发射信号，使得数据或指令看起来在近期会使用.

### vld

已记

## 运算指令



### `subs`

减法运算

### `add`

加法运算

## 向量指令

### `fmla`

float point fused multiply-add to accumulator.

$$
a = a + b \times c
$$

### `vmla`

第一个操作数向量与第二个操作数向量对应相乘，目的寄存器与结果向量对应相加。

### `vand`

向量按 bit 做与运算

### `ext`

从两个向量中提取向量

### `ins`

在一个向量中插入另一向量中的元素

### `trn1`

与 `trn2` 转置向量

## 跳转指令

### `bne`

当状态码zero 不为 0 时做跳转

### `beq`

当 状态码 zero 为 0 时跳转