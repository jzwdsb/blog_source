---
layout: post
category: matlab
date: 2018-02-04
title: matlab与神经网络 (2)
description: 径向基网络
tags: [matlab, 神经网络]
---

## 定义及特点

　　径向基网络是一种局部逼近网络，对于每个训练样本，他只需要对少量的权值和阈值进行修正，因此训练速度快

### 神经元模型

![RBFnrual](/downloads/RBFnerual.png)

　　其输出表达式(~~算了看不懂~~)为

$$
a = f(\mid \mid W - p \mid \mid \cdot b) = radbas(\mid \mid W - p \mid \mid \cdot b)
$$

## 径向基网络的创建与学习过程

1. 以 newrbe, newrb 创建径向基网络

## 其他径向基神经网络

1. 泛化回归神经网络，以 newgrnn 创建
2. 概率神经网络，以 newpnn 创建

```matlab
%% 曲线拟合
clear all;
... ...
t1 = clock;
net = newff([-1 1], [15 1], {'tansig' 'purelin'}, 'traingdx', 'learngdm');
net.trainParam.epochs = 2500;
net.trainParam.goal = 0.001;
net.trainParam.show = 10;
net.trainParam.lr = 0.05;

net = train(net, p, t);
datet = etime(clock, t1);
save net net;

p = -1:.1:0.9;
t = [-0.832 -0.423 -0.024 0.344 1.282 3.456 4.02  3.232 2.102 1.504 ...
      0.248  1.242  2.344 3.262 2.052 1.684 1.022 2.224 3.022 1.984];
t1 = clock;
% net = newrbe(p, t, 0.1);
net = newrb(p, t, 0.1, 0.1, 20, 5);
datat = etime(clock, t1);

save net net;

hold on;
plot (p, t, '*');
%% 仿真
load net net;
i = -1: 0.05: 0.9;
r = sim(net, i);

plot(i, r);

%% 模式分类
clear all;

% 定义输入变量和目标变量
p = [0 0 0 1 1 1 -1 -1 -1; 0 1 -1 0 1 -1 0 1 -1];
tc = [1 1 2 2 1 1 1 2 1];

t = ind2vec(tc);

% 设计 PNN
t1 = clock;
net = newpnn(p, n, 0.7);
datat = etime(clock, t);

save net net;

% 定义待测试样本输入向量
p = [0 0 0 1 1 -1 -1 -1; 0 1 -1 0 1 -1 0 1 -1];

load net net;

y = sim(net, net);
yc = vec2ind(y);


```

<!--
> 　　这些年我一直提醒自己一件事情，千万不要自己感动自己．大部分人看似的努力，不过是愚蠢造成的．什么熬夜看书到天亮，连续几天只睡几个小时，没多久就放假了，如果这些东西也值得夸耀，那么富士康流水线上任何一个人都比你努力多了．人难免天生有自怜的情绪，唯有时刻保持清醒，才能看清真正的价值在哪里．
-->