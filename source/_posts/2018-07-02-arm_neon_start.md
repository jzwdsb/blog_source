---
title: ARM Neon 指令入门
date: 2018-07-02
category: ARM
tags: [ARM, Neon]
---

## Neon 基础

Neon 是适用于 ARM 系列处理器的一种 SIMD(Single Instruction Multiple Data)扩展结构。Neon 有自己的执行管道和寄存器组。也就是说 Neon 和 ARM 指令是各自独立执行的。 Neon 的寄存器有

- 32 个 64 位的寄存器(D0-D31)
- 16 个 128 位的寄存器(Q0-Q15), ARM64 中有 32 个 128 位的寄存器

实际上 D 寄存器和 Q 寄存器是重叠的。如下图所示。两个 D 寄存器对应一个 Q 寄存器。

![neon_register](/image/neon_register.jpg)

在 ARM64 中，虽然 Q 寄存器有 32 个，但是 D 寄存器也只有 32 个，也就是说 Q0 只和 D0 重叠，Q1 只和 D1 重叠。

Neon 的作用在于使用向量化技术加速计算。

### 关键概念

- 数据类型(data type)
    Neon 寄存器能存放大部分的基本数据类型，如 `int8`, `int16`。一个寄存器能存储的数据个数与数据大小和寄存器大小有关。
- 向量(vector)
    在 Neon 中一个寄存器可以看做一个向量。如 64bits 的寄存器 D0,存储 4 个 `int16` 元素，就可以看做 length 为 4 的向量。
- 管道(lang)
    在上面 D0 存储 4 个 `int16`, 那么 D0 就有四个通道, pipe0 到 pipe3, pipe0 在 D0 的低 bit 位.

### 使用方法

#### intrinsics(内部函数)

使用 intrinsics 不如使用内联汇编效率高。但是使用 intrinsics 较为简单且易于维护。这些函数在编译时会直接转化为 Neon 的汇编指令

```C++
#include <arm_neon.h>
uint32x4_t double_elements(uint32x4_t input)
{
    return (vaddq_u32(input, input));
}
```

#### 使用开源库

基于 Neon 的开源库如 Project Ne10, OpenMAX DL.

#### 内联汇编

直接在 C/C++ 代码中内联汇编，但是可移植性差，且难度较高

#### 编译器产生向量化指令

可以通过添加一些编译选项的方法使能向量化编译，但是对于复杂算法编译器的效果较差

## NEON 内部函数

### 数据类型

#### 基本数据类型

- 64 位
    `int8x8_t`, `int16x4_t`, `int32x2_t`, `int64x1_t`,
    `uint8x8_t`, `uint16x4_t`, `uint32x2_t`, `uint64x1_t`,
    `float32x2_t`, `float64x1_t`(少见)
- 128 位
    `int8x16_t`, `int16x8_t`, `int32x4_t`, `int64_2_t`,
    `uint8x16_t`, `uint16_8_t`, `uint32x4_t`, `uint64x4_t`,
    `float32x4_t`

#### 结构化数据

差不多每种基本数据类型都有其对应的结构化数据类型

`int8x8x2_t`，`int16x4x2_t`，`uint8x16x2_t`，`uint16x8x2_t`

`int8x8x3_t`，`int16x4x3_t`，`uint8x16x3_t`，`uint16x8x3_t`

`int8x8x4_t`，`int16x4x4_t`，`uint8x16x4_t`，`uint16x8x4_t`

### 函数分类

Neon 内部函数分为以下几类，使用时要包含 `arm_neon.h` 头文件

- 初始化寄存器
- 从内存加载数据到 neon 寄存器
- 将 Neon 寄存器数据存储到内存
- 直接从 Neon 寄存器获取某个通道的值
- 直接设置 Neon 寄存器某个通道的值
- 寄存器数据重排
- 加减法
- 乘法
- 乘加组合运算
- 乘减组合运算
- 取整
- 比较运算
- 绝对值
- 取最大最小值
- 取倒数
- 平方根倒数
- 移位运算
- 取负数
- 按位运算
- 统计
- 数据类型转换

## 实例代码

```C++
void first_example_c(unsigned char  *src, unsigned char  *dst, int size, unsigned char  factor)
{
    for (int i = 0; i < size; i++)
    {
        dst[i] = src[i] * factor;
    }
}

void first_example_neon(unsigned char  *src, unsigned char  *dst, int size, unsigned char  factor)
{
    int main_loop = size / 8; //一个循环处理8个数据，则需要 size / 8个循环
    uint8x8_t factor_vct = vdup_n_u8(factor); //将系数factor装载入neon寄存器
    for (int i = 0; i < main_loop; i++)
    {
        uint8x8_t src_vct = vld1_u8(src);//将源数据装载入neon寄存器
        uint8x8_t dst_vct = vmul_u8(src_vct, factor_vct);  //执行乘法操作，且将结果放入dst_vct寄存器中
        vst1_u8(dst, dst_vct); //将dst_vct寄存器中的结果放回内存中
        src += 8， dst += 8; //改变地址，指向下个循环要处理的数据。
    }
}
```