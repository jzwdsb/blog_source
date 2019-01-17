---
title: CNN 的基础运算算子
category: CNN
date: 2018-08-20 15:00
tags: [CNN]
mathjax: true
---

这里记下卷积神经网络的常用算子，并做简单的分类。主要~~参考~~抄自[知乎](https://zhuanlan.zhihu.com/p/32711259)

# 深度神经网络计算

## 计算层

这部分算子是深度卷积神经网络的核心，用于将输入的神经元激活值与突触连接强度(权重)进行积分求和，得到新的神经元的膜电位。ヽ(*。>Д<)o゜
根据是否滑窗，是否具有时序结构，可分为如下几种算子，其中 FC(全连接层) 是多层感知机(MLP)的基础，Conv 和 FC 是深度卷积神经网络的基础。RNN, GRU 和 LSTM 是带有时序结构的神经网络模型，主要用于非静态图像的场合，例如语音，文字，视频等。

- Convolution
- Convolution Transpose
- Fully Connection
- RNN
- LSTM
- GRU

## 池化层

池化层主要用于尺度变换，提取高维特征，主要分为三种池化
局部池化，在图像维度上几个相邻的点被缩减为一个输出点，在 Channel 维度上不变。包括平均池化(AveragePool), 最大值(MaxPool), 范数(LpPool)。主要用于图像的尺寸变换。
全局池化，此时一个 Channel 的所有数据点缩为一个点，因此有几个 Channel 就输出几个数据点。此种类型主要用于深度卷积神经网络中卷积部分与 FC 部分的链接。
ROI-Pooling, 主要用于 Faster-RCNN 等检测识别网络中对感兴趣区域进行尺度归一化，从而输入到识别网络进行识别处理。

- Average Pool
- Max Pool
- Lp Pool
- Global Average Pool
- Global Lp Pool
- Global Max Pool
- Max Roi Pool

## 批数据归一化层

归一化层作为一个特殊层，可用于数据的归一化，提高神经网络的性能，降低训练时间。对于带有残差的神经网络非常重要。
目前的高性能网络大多带有归一化层，而绝大多数都会采用 Batch Normalization(BN)。BN 的前向操作并不复杂，但是反向比较复杂，因此用于训练的 BN 需要加入更多的子层。
归一化层的计算方式有 Instance Normalation 和 基于范数的归一化。

- Batch Normalization
- Lp Normalization
- Instance Normalization
- LRN
- Mean Variance Normalization

## 数据归一化

将数据进行归一化处理，通常用于输出层的归一化。

- SoftMax
- LogSoftMax
- HardSoft

## 其他计算层

Dropout 随机扔掉一些通路，可以用于防止过拟合。
Embedding 用于将词转换为高维表达，是文本的预处理的主要步骤。
GRUUnit 是个实验性函数，功能类似于 GRU 的激活层。

- Dropout
- Embedding
- GRUUnit

# 基础 Tensor 运算

## 逐元素运算(element-wise)类

终于知道 element-wise 怎么翻译了

这个类别包含了 Tensor 的一些基础运算，由于输出的数据点只跟对应的哪一个输入的数据点有关，因此可以称为 element-wise 运算，这类运算与输入的数据的维度和结构无关，可以等价的认为是一维向量运算的 Tensor 等效表示。
由于输入数据可能是各种维度，也可以是标量，因此这里的操作都是维度兼容的。
一类特殊情况是一个是向量，一个是标量，scalar-tensor 或者 tensor-scalar

|        |      |            |         |          |
| ------ | ---- | ---------- | ------- | -------- |
| And    | Or   | Not        | Xor     | Max      |
| ArgMax | Min  | ArgMin     | Greater | Less     |
| Equal  | Abs  | Add        | Sub     | Mul      |
| Div    | Neg  | Reciprocal | Sqrt    | Log      |
| Exp    | Pow  | Scale      | Affine  | Identity |
| Clip   | Cast | Ceil       | Floor   |          |

## Tensor/矩阵处理类

这部分操作是对整个 Tensor 的数据进行的，即输出可能关系到 Tensor 中的不止一个数据。包括求和，求平均，通用矩阵运算(Gemm), 矩阵乘法，图像缩放等。
其中 Gemm 是矩阵处理的通用表达形式, 即 $Y=\alpha\times A\times B + \beta \times C$, 其中 A 为 $M\times K$ 维，B 为 $K\times N$ 维，C 和 Y 为 $M\times N$ 维。

- Sum
- Mean
- Gemm
- MatMul
- ImageScaler

## 激活和非线性函数

激活函数提供了神经网络的非线性拟合能力，不同的激活函数具有各自的性能特点。由于 ReLU 简单且性能较好，因此一般图像处理算法采用 ReLU 函数。而 Sigmoid 和 Tanh 在 LSTM/GRU/RNN 中较为常见。这些函数的计算也是 Element-wise 方式，但是公用较为特殊

|                 |            |                    |           |             |
| --------------- | ---------- | ------------------ | --------- | ----------- |
| ReLu            | Sigmoid    | Tanh               | Elu       | Selu        |
| PRelu           | SoftPlus   | Softsign           | LeakyReLu | HardSigmoid |
| ThresholdedRelu | ScaledTanh | ParametricSoftPlus |           |             |

## 随机数和常数

这些操作用于产生数据，包括正态随机产生，均匀随机产生，常数等。

- RandomNormal
- RandomNormalLike
- RandomUniform
- RandomUniformLike
- Constant

# Tensor 变换

这部分算子不会改变 Tensor 的数据，只会对数据的位置和维度进行调整

## 分割组合算子

这部分可以将多个 Tensor 合并为一个，或者将一个拆分为多个。可以用于分组卷积等。

- Concat
- Split

## 索引变换

索引变换包括 Reshape, 矩阵转置，空间维度和 Feature Map 互换等。可以认为是数据排布关系的变化。
Flatten 将输入 Tensor 平展为一维向量，Squeeze 去掉输入 Tensor 中维度为 1 的维，用于压缩 Tensor 的维数。
Flatten 和 Squeeze 可以认为是 ReShape 的特殊情况

- Transpose
- SpaceToDepth
- DepthToSpace
- Reshape
- Flatten
- Squeeze

## 数据选取

这部分操作可以根据维度参数，边框或者脚标矩阵参数选取 Tensor 的部分数据，或者对 Tensor 的数据进行复制拓展

- Slice
- Gather
- Tile
- Crop

## 数据填充

数据填充分为边缘补 0, 常数填充和拷贝。

- Pad
- ConstantFill
- GivenTensorFill