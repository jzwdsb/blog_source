---
title: BatchNorm 原理
date: 2018-08-06
category: CNN
tags: [CNN, ncnn]
mathjax: true
---

关于 BatchNorm 层的原理可以详见此[论文](https://arxiv.org/pdf/1502.03167.pdf)

## BatchNorm 解决的问题

深度卷积神经网络的训练主要是每一层神经元学习本层的输入数据的分布。但是输入数据的分布会随着上一层的计算发生变化，称为 `internal covatiate shift`. 给本层的学习带来了困难，在 `BatchNorm` 之前是通过微调学习率和初始化参数实现，学习速度很低。
引入 `BatchNorm` 通过对数据规范化处理避免这个问题。

## 原理

### 输入

小批量数据: $B = \{x_1, ...m \}$
需要学习的参数: $\gamma, \beta$

### 输出

$y_i = BN_{\gamma,\beta}(x_i)$

### 算法

$$
\mu_B \leftarrow \frac 1 m \sum_{i=1}^{m}x_i \quad \text{mini-batch mean} \\\
\sigma_{B}^{2} \leftarrow \frac 1 m \sum_{i=1}^{m}(x_i - \mu_B)^2  \quad \text{mini-batch variance} \\\
\hat{x}_i \leftarrow \frac {x_i - \mu_B} {\sqrt{\sigma_B^2 + \epsilon}} \quad \text{normalize} \\\
y_i \leftarrow \gamma \hat{x}_ {i}+\beta\equiv {BN_{\gamma,\beta}(x_i)} \quad \text{scale and shift}
$$

- 求出此批量数据的均值
- 求出此批量的方差
- 归一化处理
- 引入缩放和平移变量 $\gamma$ 和 $\beta$

BatchNorm 降低了数据的绝对差异，突出了相对差异，在分类任务上效果明显。