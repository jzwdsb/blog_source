---
layout: post
category: 统计学习方法
date: 2018-01-29
title: 感知机
description: 定义及算法
mathjax: true
tags: [机器学习, 感知机]
---

　　感知机(perceptron) 是二类分类的线性分类模型，输入为实例的特征向量，输出为示例的类别，取 -1 和 +1 二值．

## 定义
　　假设输入空间(特征空间)是 $\chi \subseteq R^n$，输出空间是 $y = \{+1, -1\}$.
输入 $x \in \chi$, 表示实例的特征向量，对应于输入空间(特征空间)的一个点;
输出 $y \in \cal Y$表示实例的类别. 由输入空间到输出空间有如下函数

$$ f(x) = sign(w \cdot x + b) \tag{1}$$

　　以上函数称为感知机. 其中，w 和 b 为感知机模型参数，　$w \subseteq R^n$叫做权值(weight), 或权值向量(weight vector),　$b \subseteq R$$叫做偏置(bias)，$w \cdot x$表示 w 和 x的内积，
sign是符号函数

$$
sign(x) =
\begin{cases}
+1, & \text{x $\ge$ 0} \\
-1, & \text{x $\lt$ 0} \\
\end{cases} \tag{2}
$$

## 损失函数

　　给定一个训练数据集

$$T = \{(x_1, y_1), (x_2, y_2), ..., (x_N, y_N)\} $$

　　其中, $x_i \in \chi = R^n, y_i \in \cal Y = \{-1, 1\}, i = 1, 2, ..., \mit N$. 感知机$sign(w \cdot x + b)$学习的损失函数定义为
$$L(w, b) = -\sum_{x \in M}y_i(w \cdot x_i + b) \tag{3}$$

其中, M 为误分类点的集合．

## 算法

　　感知机学习问题可转化为求解损失函数式的最优化问题，最优化的方法是随机梯度下降法.

### 感知机学习算法的原始形式

　　给定一个训练数据集

$$T = \{(x_1, y_1), (x_2, y_2), ..., (x_N, y_N)\} $$

　　其中, $x_i \in \chi = R^n, y_i \in \cal Y = \{-1, 1\}, i = 1, 2, ..., \mit N$,
求参数w, b，使得其为以下损失函数极小化问题的解

$$ \underset{w, b}{min}L(w, b) = - \sum_{x_i \in M} y_i(w \cdot x_i + b) \tag{3}$$

其中 M 为误分类点的集合

　　感知机学习算法是误分类驱动的，具体采用随机梯度下降法(stochastic gradient descent).首先任意选取一个超平面$w_0, b_0$, 然后用梯度下降法不断的极小化目标函数．<br>
　　假设误分类点集合M是固定的，那么损失函数 L(w, b)的梯度由
$$ \nabla_wL(w, b) = - \sum_{x_i \in M} y_ix_i \tag{4}$$
$$ \nabla_bL(w, b) = - \sum_{x_i \in M} y_i \tag{5}$$
给出
　　随机选取一个误分类点$$(x_i, y_i)$$，对w, b进行更新
$$ w \leftarrow w + \eta y_i \tag{6}$$
$$ b \leftarrow b + \eta y_i \tag{7}$$
　　其中$\eta(0 \leq \eta \leq 1)$是步长，在统计学习中又称为学习率(learning rate).

#### 算法描述

　　输入: 训练数据集 $T = \{(x_1, y_1), (x_2, y_2), ..., (x_N, y_N)\}$, 其中$x_i \in \chi = R^n, y_i \in \cal Y = \{-1, 1\}, i = 1, 2, ..., \mit N$，学习率$n \leq 1$;
　　输出: w, b;　感知机模型$f(x) = sign(w \cdot x + b)$.

1. 选取初值$w_0, b_0$
2. 在训练集中选取数据$(x_i, y_i)$
3. 如果$y_i(w \cdot x + b) \leq 0$

    $$ w \leftarrow w + \eta y_ix_i \tag{8}$$
    $$ w \leftarrow b + \eta y_i \tag{9}$$
4. 转至２，直到训练集中没有误分类点．
　　当一个实例点被误分类，即误分类，即位于超平面的错误一侧时，则调整w, b的值，使分离超平面向该误分类点的一侧移动，以减少
该误分类点与超平面的举例，直到超平面越过该误分类点使其被正确分类

### 感知机学习算法的对偶形式

　　对偶形式的基本想法是，将 w 和 b 表示为示例 $x_i$和标记 $y_i$的线性组合的形式，通过求解其系数求得 w 和 b．不失
一般性，在原始形式中可假设初始值 $w_0$ 和 $b_0$, 对误分类点$(x_i, y_i)$通过

$$ w \leftarrow w + \eta y_i x_i \tag{10}$$

$$ b \leftarrow b + \eta y_i \tag{11}$$

逐步修改 w, b, 修改 n 次，则 w, b 关于 $(x_i, y_i)$ 的增量分别是 $\alpha = n_i \eta$. 这样，从学习过程不难看出，
最后学习到的w, b可以分别表示为

$$ w = \underset{i=1}{\sum^N} {\alpha}_i y_ix_i \tag{12}$$

$$ b = \underset{i=1}{\sum^N} {\alpha}_iy_i \tag{13}$$

这里, ${\alpha}_i \geq 0, i=1, 2, ..., \mit N$, 当$\eta = 1$时，表示第i个实例点由于误分而进行更新的次数，实例点更新次数越多，
意味着它举例分离超平面越近，也就越难正确分类．这样的实例对学习结果影响最大

<!--
> 　　有许多我当年以为能在心中长存不衰的东西都变得残破不堪，新的事物继而兴起，衍生出许多我当年意想不到的悲欢，
> 同样，旧的事物都变的难以理解了．　　　　　　　　　　　　　----- 追忆似水年华
-->