---
layout: post
category: matlab
date: 2018-02-14
title: 数模随笔
description: 这次数学建模用到的方法
tags: [matlab]
---

　　胡乱建模大赛忙完了，四天的努力也造出了一篇学术垃圾，记下中间用到的方法

1. 数据归一化处理(mapminmax)
2. 聚类(kmeans)
3. 回归分析(regress)
4. 拟合
5. 数据可视化(plot)
6. excel表处理

## 数据归一化

　　因为各项数据量纲不同，所以将所有数据做归一化处理以方便做聚类，这个模型中的归一化处理使用的是 matlab nerual network 工具箱中的 mapminmax.

### mapminmax

　　将矩阵中的元素根据行的最大值和最小值映射到 [-1, 1]\(默认)之间.<br>
　　公式如下

$$y = (y_{max} - y_{min}) \times (x - x_{min}) / (x_{max} - x_{min}) + y_{min}$$

　　其中 $$y_{max}, y_{min}$$ 是调用时输入的上下限，默认是 [-1, 1], $$x_{max}, x_{min}$$ 是 $$y$$ 所在行中的最大值和最小值．

　　调用方式如下

```matlab
[Y, PS] = mapminmax(X, YMIN, YMAX);
[Y, PS] = mapminmax(X, FP);
Y = mapminmax('apply', X, PS);
X = mapminmax('reverse', Y, PS);
dx_dy = mapminmax('dx_dy', X, Y, PS);
```

1. **输入**: 矩阵 X, 可选参数 YMIN, YMAX; <br> **输出**: 归一化结果 Y, 原始数据到结果的映射记录 PS.
2. **输入**: 矩阵 X, 可选参数 FP(将映射区间作为结构体 FP.ymin, FP.ymax); <br> **输出**: 归一化结果 Y, 原始数据到结果的映射记录 PS
3. **输入**: 调用方式 'apply', 输入矩阵 X, 映射记录 PS, 即归一化处理按照 PS 中记录的方式映射.
4. **输入**: 调用方式 'reverse', 输入矩阵 X, 映射记录 PS, 即由已归一化数据得到原始数据，必须要有映射记录. <br> **输出**: 原始数据.
5. **输出**: 返回逆向导数.~~官网就这一句话，资料不全，不常用~~

## 聚类

　　原始数据中总共有 1500 多项指标可用于评估，去除数据严重不全的指标也有 1000 多种. 为了减少数据的处理量，参考评估结果中总共有 12 个方面，故对归一化后的数据做聚类处理. 这里使用的是matlab statistics and machine learning 工具箱中的 kmeans.<br>
　　得到聚类结果后每个簇大概只有相近的 50 到 100 种指标，从每一簇中人为找出最具代表意义的指标，聚类工作减少了人工筛选的数量.~~并没有~~<br>
　　~~我感觉这样做毫无道理，类别相同的无非也就是数据的高维空间距离相近而已，组里的大佬说什么就是什么了-_-~~

### kmeans

　　kmeans 算法详见 [Andrew Ng 介绍](http://cs229.stanford.edu/notes/cs229-notes7a.pdf). 这里简单说明下<br>
　　给出训练集 $$\{x^{(1)}, ..., x^{(m)}\}$$, 这里希望将这些输入数据可以区分为几个内聚的簇．<br>
　　聚类(clustering)与分类(classification)不同在于，聚类算法是无监督学习(unspurvised learning)的一种．

　　聚类算法的简单理解是将矩阵中要聚类的行向量作为一个高维空间的坐标，点间的距离使用欧几里德公式得出．聚类算法旨在找出在空间中的分布相近的簇．簇的数量由输入决定.

　　调用方式如下

```matlab
idx = kmeans(X, k);
idx = kmeans(X, k, Name, Value);
[idx, c] = kmeans(__);
[idx, C, sumd] = kmeans(__);
[idx, C, sumd, D] = kmeans(__);
```

1. **输入**: $$n \times p$$ 矩阵 X, 聚类个数 k; <br> **输出**: $$n \times 1$$ 列向量 idx, 输入矩阵中每个行向量的观测值，范围在 1 到 k 之间.
2. **输入**: $$n \times p$$ 矩阵 X, 聚类个数 k, Name, value 配对参数指定的聚类选项. <br> **输出**: 列向量 idx, 每个行向量的观测值.
3. 返回 $$n \times p$$ 矩阵中质心位置
4. 返回 $$k \times 1$$ 向量中该簇点到质心距离总和 sumd.
5. 返回每个点到 $$n \times p$$ 矩阵中每个质心距离 D.

## 回归分析

　　已有 12 项指标用于脆弱性评估，题目已给出国家的脆弱性指数，找出指标与脆弱性指数的函数对应关系．<br>
　　于是接下来做回归分析.~~其实可以把数据和标签喂给神经网络，但是这样论文就没什么可写了~~<br>
　　由于存在 12 项输入，输出只有一个，所以采用的回归方法是多元线性回归，高次的拟合在这里并不必要．<br>

　　多元线性回归模型的一般形式

$$Y_i = \beta_{0} + \beta_1 X_{1i} + \beta_2 X_{2i} + ... + \beta_k X_{ki} + \mu_i, i = 1, 2, ...,n $$

　　多元线性回归的计算模型

$$Y = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + ... + \beta_k x_k + \epsilon, \epsilon \in N(0, \delta^2) $$

　　参数估计同一元线性回归相同，也是在要求误差平方和最小的前提下，用最小二乘法或极大似然估计法求解参数．

### regress

　　matlab 中多元线性回归的实现是 statistics and machine learning 工具箱中的regress.

　　调用方式如下

```matlab
b = regress(y, X);
[b, bint] = regress(y, X);
[b, bint, r] = regress(y, X);
[b, bint, r, rint] = regress(y, X);
[b, bint, r, rint, stats] = regress(y, X);
[b, bint, r, rint, stats] = regress(y, X, alpha);
```

#### 输入

1. 列向量 y, 观察到的响应，对应于 n 个观测值．
2. 矩阵 X, $$n \times p$$矩阵，对应于 n 个观测值，p 对应于拟合的变量个数．
3. 第一列必须全为 1, 用于拟合常量.
4. 可选参数 alpha, 使用 $$100\% \times (1 - alpha)$$ 置信度计算 bint, rint.

#### 输出

1. $$n \times 1$$ 列向量 b, 系数估计值.
2. $$n \times 2$$ 矩阵 bint, 每一行两个值表示系数估计值的置信区间．
3. $$n \times 1$$ 列向量 r, 残差．
4. $$n \times 2$$ 矩阵 rint, 如果 rint(i, :) 观测间隔不包含 0, 表明残差为异常值．
5. $$1 \times 4$$ 行向量 stats, 顺序包含 $$R^2$$ 统计量，F 统计量及其 p 值，以及误差方差的估计．

## 拟合

　　没什么可说的，唯一值得说的是应该用归一化后的数据还是原数据，这里使用的数据应该和回归分析时所用的数据保持一致．<br>
　　新增的测试数据如果做归一化处理每一次还要将整个训练数据一起运算，其实是应该用原数据做回归分析和预测.<br>
　　计算公式如下，两个向量的内积

$$F = \sum_{i = 1}^{13} b_i x_i$$

　　这里 b 是回归分析时所得出的各项系数，第一个为常数项； x 为新数据向量，包含 12 个指标参数，第一个必须为常数 1.

## 数据可视化

　　一篇优秀的论文是离不开各种好看的彩图的.~~大误~~

　　论文需要的图像大都可以用 plot, surf 两个绘图专有函数实现，调用方式非常多，是个大话题．

## excel 表处理

　　由于从 [word bank](https://data.worldbank.org.cn/)得到的数据全为 excel 表格，所以熟悉下 excel 还是很有必要的

<!--
> 　　我控告您无视爱情，<br>　　一味逃避，<br>　　唯唯诺诺，<br>　　我判处您终生孤寂．
-->