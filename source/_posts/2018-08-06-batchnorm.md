---
title: BatchNorm 原理
date: 2018-08-06
category: CNN
tags: [CNN, ncnn]
mathjax: true
---

关于 BatchNorm 层的原理可以详见此[论文](https://arxiv.org/pdf/1502.03167.pdf)

## BatchNorm 解决的问题

深度卷积神经网络的训练主要是每一层神经元学习本层的输入数据的分布。但是输入数据的分布会随着上一层的计算发生变化，称为 `internal covatiate shift`. 显然给本层的学习带来了困难，`BatchNorm` 就是通过对数据规范化处理避免这个问题。

## 原理

### 输入数据

> TODO 明天接着写