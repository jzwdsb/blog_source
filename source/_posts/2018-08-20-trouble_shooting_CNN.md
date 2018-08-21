---
title: CNN 的常见问题分析和解决方法
category: CNN
date: 2018-08-20 11:00
tags: [CNN]
---

记下 CNN 中 troubleshooting 的常用方法，原文见于[链接](https://gist.github.com/zeyademam/0f60821a0d36ea44eef496633b4430fc)

## 分析之前

- 使用合适的日志和易于理解变量名
  在 tensorflow 中，可以使用 tensorboard 来可视化计算图。
- 确定网络连接正确
  保证网络之间的输入输出关系正确
- 使用数据增强技术
  并不总是适用，常用方法有镜像，旋转，增噪。
- 确保正则化操作不会一直损失函数中的其他因子
  关掉正则化操作，观察函数损失的数量级的顺序，适当的调整正则化操作的权重，确保正则化后函数损失增大
- 尝试在一个小数据集上过拟合
- 当在小数据集上过拟合后，寻找合适的学习率
- 执行梯度检查

## 函数损失没有提高

- 确保使用了合适的损失函数并且优化了正确的 tensor
  常用的损失函数可见[链接](https://en.wikipedia.org/wiki/Loss_functions_for_classification)
- 使用正确的优化器
  常用的优化器可见[链接](https://keras.io/optimizers/)
- 确保训练的变量正在优化
  使用 tensorflow 的话，可能需要查看 tensorflow 的直方图，或者写一个脚本，计算出在不同训练时期的每个 tensor
- 调整初始学习率，实现合适的学习率调度
- 确保没有过拟合

## 变量没有在训练

- 确保它被 tensorflow 当做训练变量
- 确保没有梯度耗散
- 确保 ReLus 在工作

## 梯度耗散/梯度爆炸

- 使用数据增强技术
- 实现 dropout 层
- 增强规范化
- 实现 Batch Normalization 层
- 实现基于正确性的终止
- 如果失败，使用一个更小的网络

## 其他

- 使用加权损失函数
- 改变网络的架构
- 使用更为协调的模型
- 将 max/average pooling 替换为 strided convolutions
- 执行超参数的查询
- 替换随机数种子
- 如果不成功，需要更多的训练数据