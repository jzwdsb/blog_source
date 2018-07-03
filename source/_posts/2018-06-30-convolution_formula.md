---
title: CNN 公式推导
description: 基本就是抄自知乎了 -_-
category: 神经网络
tags: [Andrew Ng, UFLDL, CNN]
mathjax: true
---

## 矩阵卷积

在图像处理中，我们常会用卷积来提取图像(image)的特征(feature), 矩阵卷积有如下两种

假设 $X$ 是 $m \times m$ 矩阵，$K$ 是 $n \times n$ 型矩阵，$K_{rot}$ 由 $K$ 旋转 $180^o$ 得到，

- 全卷积(full convolution)
    全卷积的定义式如下
    $$
    z(u, v) = \sum_{i = - \infty}^{+\infty} \sum_{j = - \infty}^{+ \infty} x_{i, j} \cdot k_{u-i, v-j}
    $$
- 有效卷积(vaild convolution)
    有效卷积的定义如下
    $$
    z(u, v) = \sum_{i = -\infty}^{+\infty}\sum_{j=-\infty}^{+\infty} x_{i + u, j + v} \cdot k_{rot_ {i, j}} \cdot \chi(i, j) \\\
    \chi (i, j) =
    \begin{cases}
    1, 0 \leq i, j \leq n \\\
    0,  \text{others}
    \end{cases}
    $$

## 通过正向传播(forward propagation)计算激活值(activation)

- 卷积层(上一层为输入层)
    在卷积层数据都是以三维数据存在的。假定输入层是 $l - 1$ 层，它的输出(对应于 $l$ 层的输入)的特征图为 $X^{l-1}(m \times m)$, 特征对应的卷积核是$K^l (n \times n)$, 给每一个输入单元加上一个偏置单元(bias term)$B^{(l)}$, 卷积层的输出矩阵为 $M^{(l)}((m - n + 1) \times (m - n + 1))$ 阶矩阵，计算公式如下
    $$
    z_{u, v}^{(l)} = \sum_{i=-\infty}^{\infty} \sum_{j=-\infty}^{\infty} x_{i+u, j+v}^{(l-1)}\cdot k_{rot_{i, j}}\cdot \chi (i, j) + b^{(l)} \\\
    \chi(i, j) =
    \begin{cases}
    1， 0 \leq i, j \leq n \\\
    0, others
    \end{cases} \\\
    a_{u, v}^{(l)} = f(z_{u, v}^{(l)})
    $$
- 子采样层(sub-sample layer)
    通过卷积操作我们得到了关于输入数据的特征图，我们希望我们能够利用所有的特征图训练我们的分类器，但这样存在输入数据过大，容易发生过拟合的问题。
    可以采用聚合统计的方法避免这个问题，通过计算区域中最大值的像素或者计算平均数的方法降低输入数据的维度。这种聚合操作就是池化操作(pool)。上面两种方法分别对应于平均池化(average pooling) 和最大值池化(max pooling)
    以平均池化为例，每个单元的权重为 $\beta^{(l+1)}$,滑动步长为 $r$,每一项卷积操作之后都加上一个偏置单元 $b_{i,j}^{(l+1)}$, 子采样层的输出为 $a_{i,j}^{(l+1)}(\frac {m-n+1}r\times\frac{m-n+1}r)$阶矩阵，计算公式为
    $$
    z_{i,j}^{(l+1)}=\beta^{(l+1)}\sum_{u=ir}^{(i+1)r-1}\sum_{v=jr}^{(j+1)r-1}a_{u,v}^{(l)}+b^{(l+1)}\\\
    a_{i,j}^{(l)}=f(z_{i,j}^{(l+1)})
    $$
- 卷积层(上一层为子采样层)
    如果子采样层后接的是卷积层，计算方式和多层神经网络描述的相同，输出 $A^{(l+2)}((\frac{m-n+1}r-n+1)\times(\frac{m-n+1}r-n+1))$ 阶矩阵为
    $$
    z_{(u,v)}^{(l+2)}=\sum_{i=-\infty}^{\infty}\sum_{j=-\infty}^{\infty}a_{i+u,j+v}^{(l+1)}\cdot k_{i,j}^{(l+1)}\cdot\chi(i,j)+b^{(l+2)}\\\
    a_{u,v}^{(l+2)}=f(z_{u,v}^{(l+2)})
    $$

## 通过反向传播计算残差(error term)