---
title: 关于加入 bias 优化的想法
description: 单通道，$1 \times 1$ 卷积核
category: CNN
date: 2018-07-06
mathjax: true
---

## 源码

单通道，卷积核大小为 $1 \times 1$, 在卷积核大小 $1\times1$ 的情况下，输出仅仅是对图像中的每一个像素乘以一个系数。

```C
asm volatile(
    "pld        [%2, #256]          \n"
    "vld1.f32   {d4-d7}, [%2 :128]! \n"
    "0:                             \n"
    "pld        [%1, #256]          \n"
    "vld1.f32   {d0-d3}, [%1 :128]  \n"
    "vmla.f32   q0, q2, %q6         \n"
    "vmla.f32   q1, q3, %q6         \n"
    "pld        [%2, #256]          \n"
    "vld1.f32   {d4-d7}, [%2 :128]! \n"
    "subs       %0, #1              \n"
    "vst1.f32   {d0-d3}, [%1 :128]! \n"
    "bne        0b                  \n"
    "sub        %2, #32             \n"
  : "=r"(nn),     // %0
    "=r"(outptr), // %1
    "=r"(r0)      // %2
  : "0"(nn),
    "1"(outptr),
    "2"(r0),
    "w"(_k0)      // %6
  : "cc", "memory", "q0", "q1", "q2", "q3"
                    );
```

## bias

这里 `bias` 的加入实在代码中循环开始前使用 `Mat::fill` 将 `output` 初始化为 `bias` 实现。

现在优化为在卷积计算过程后中使用 Neon 向量运算指令加到 `output` 中，理论上运算速度会快一点。