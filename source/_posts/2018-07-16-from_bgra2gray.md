---
title: BGRA 与 GRAY 颜色空间的转换
description: bgra2gray 和 gray2bgra
category: 图像处理
tags: [Neon, ARM]
date: 2018-07-16
mathjax: true
---

## 原理

在图像处理中，会需要从一个颜色空间转换到另一个颜色空间。 OpenCV 中关于颜色空间转换的公式可见[链接](https://docs.opencv.org/3.1.0/de/d25/imgproc_color_conversions.html)。`GBRA` 到 `GRAY` 空间的转换与 `RGB[A]` 到 `GRAY` 的计算公式并无不同

### bgra2gray

- 浮点计算公式
  $$
  Y \leftarrow 0.114 \times B + 0.587 \times G + 0.299 \times R
  $$

- 整数移位计算公式
  $$
  Y \leftarrow (1868 \times B + 9617 \times G + 4899 \times R) >> 14
  $$

### gray2bgra

$$
B \leftarrow Y \\\\
G \leftarrow Y \\\\
R \leftarrow Y \\\\
A \leftarrow max(channelrange)
$$

## 实现

具体的实现需要一些 trick, 比如使用移位运算代替浮点运算，使用向量运算指令或者 intrinsics 防止编译器做负优化. (~~写这种代码真是心累~~)
实际上这么常用的功能一定有人已经做过，找到了一篇[博文](https://www.tuicool.com/articles/mYnaMb)(12年的，不知道有没有过时)，当然 github 上面也有几乎完全符合要求的[开源项目](https://github.com/carlj/NEON-ASM-BGRA-to-Grayscale-conversion)(ヾ(✿ﾟ▽ﾟ)ノ), 整理下方法，然后 copy and paste.

### 实现颜色空间转换的几种途径

- 使用 OpenCV 中的 `CV::cvtColor`
- 使用内联 ARM Neon 指令
- 使用 Neon intrinsics
- 多线程并行处理

比较下性能当然是最后一种最好，在使用 ARM Neon 向量运算指令的加速下，多线程并行处理，速度一般可以达到 OpenCV 的 1.5 到 6 倍以上。
博文中的并行实现是使用OpenCV 中的 `cv::paralled_for`, 这个接口将用户代码与多线程后端实现分离，使用这个接口需要实现 `cv::parallelLoopBody` 函子。
目前的项目没有使用 OpenCV, 所以使用其他方法并行化，OpenMP 应该是可以使用的。但是使用 ASM 内联汇编的话 OpenMP 无法针对汇编级代码并行化，可以试试 intrinsics 和 OpenMP.

实现时需要注意的是系数与具体公式中的系数并不相同。一般的 BGRA 图像的深度为 8 位，计算的精度要求并不高，公式中实现右移 14 位只是误差最小的描述，具体实现时的公式为

$$
Y \leftarrow (29 \times B + 150 \times G + 77 \times R) >> 8
$$

其实就是等比缩小 $2^6$ 而已。

## 注意事项

~~奇怪，项目中的灰度图的深度居然是都是 `f32`, 是我理解错了吗，从来没见过这种深度。~~
(半天后添加)这里使用的灰度图只是在神经网络作为中间计算结果，不作为显示，后续运算的特征图像素为 `float` 类型，所以这里直接将像素转为 `float` 类型。
