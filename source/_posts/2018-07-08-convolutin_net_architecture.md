---
title: 卷积网络的结构
date: 2018-07-08
category: CNN
tags: [CNN, Convolution]
mathjax: true
---

卷积神经网络的一般结构如下

- Input Layer
  保存着输入图像的原始数据
- Convolution Layer
  每个神经元都会使用各自的权重与其相连的区域做点乘，然后作为该神经元的输出。通常输出通道的个数与卷积核的数量相同
- ReLU Layer
  这一层会对每一个像素使用 $ReLU$ 函数，最简单的函数形式为 $max(0, x)$。更一般的形式为
  $$
  E =
    \begin{cases}
    E, if \\ E > threhold \\\
    E \cdot factor, if \\ E < threhold
    \end{cases}
  $$
  $max(0, x)$ 即当 $threhold,factor$ 为 0 的情况
- Pool Layer
  池化层会对其输入进行下采样，降低输入的维度，防止过拟合
- Fully Connection Layer
  会计算出对应于每个类的概率，计算结果通常为一维向量。长度为类的个数