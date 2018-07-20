---
layout: post
category: matlab
date: 2018-02-08
title: matlab 随笔
description: 模拟退火算法
tags: [matlab, 神经网络]
---

## 题目描述

~~搜索到的全是这道题，某年真题吗~~

> 　　我方有一个基地，经纬度是(70, 40). 假设我方飞机的速度为 1000 公里 / 小时．我方派一架飞机从基地出发，侦查完敌方所有目标，再返回原来的基地．在敌方每一目标点的侦查时间不计，求该飞机所花费的时间.

　　[该题数据](/downloads/base_location.txt)

## 分析

　　这是一个旅行商问题．我们依次给基地编号为 1, 地方目标依次编号为 2, 3, 4, ..., 101. 最后我方基地再重复编号为 102 (便于计算). 使矩阵 $$D = (d_{i, j}) 102 \times 102$$ 　　其中，$$d_{i,j}$$表示i, j 两点之间的距离，$$ 1 \leq i \leq 102, 1 \leq j \leq 102$$, 这里 D 为实对称矩阵．则问题可转化为从点 1 出发，走遍中间所有点，到达点 102 的一个最短路径

## 模拟退火算法

### 认识

　　模拟退火算法是用来求解最优化问题的算法，如 TSP 问题，函数最大最小值问题等．

### 描述

　　若 $$J(Y(i + 1)) \geq J(Y(I))$$, 即移动后得到更优的解，则总是接受该移动．
　　若 $$J(Y(i + 1)) < J(Y(i))$$, 即移动后得到更差的解，则以一定的概率接受该移动，并且这个概率随时间推移逐渐降低，此概率表示为

$$P(dE) = exp(\frac{dE}{kT})$$

　　由于是退火过程，所以 $$dE < 0$$, 这个公式说明了温度越高出现一次能量差 dE 的降温概率就越大；温度越低，出现降温的概率越小，由于 $$dE$$ 总是小于 0, 所以 $$P(dE)$$ 取值在 0 到 1之间．

### 伪代码

```c++
/** J(y)        状态 y 时的评价函数值
    Y(i)        当前的状态
    Y(i + 1)    下一个新的状态
    delta       用来控制降温的快慢
    T           系统的温度
    T_min       温度的下限，若温度达到 T_min, 则停止搜索
 */

while(T > T_min)
{
    dE = J(Y(i + 1)) - J(Y(i));
    if (dE >= 0)
    {
        /** 移动后得到更优的解，则总是接受此移动*/
        Y(i + 1) = Y (i);
    }else
    {
        /** dE 的值越大，则此条件成立概率越大，T 越低 exp(dE / T) 越小*/
        if (exp(dE / T) > random(0, 1))
        {
            Y(i + 1) = Y(i);
        }
    }
    /** 降温*/
    T *= delta;
    ++i;
}
```

## matlab 实现

```matlab
clear all;

load base_location.txt
ij = base_location;

x = ij(:, 1:2:8);
x = x(:);

y = ij(:, 2:2:8);
y = y(:);

sj = [x y];
dl = [70 40];
sj = [dl;sj;dl];
sj = sj * pi / 180;

d = zeros(102);

%% 计算距离
for i = 1 :101
    for j = i + 1 :102
        d(i, j) = 6370 * acos(cos(sj(i, 1) - sj(j, 1)) * cos(sj(i, 2)) ...
                * cos(sj(j, 2)) + sin(sj(i, 2)) * sin(sj(j, 2)));
    end
end

%% 已知距离矩阵 d 为对称矩阵
d = d + d';
S0 = [];
Sum = inf;
rand('state', sum(clock)); % 定义一个随时间变化的初值

for i = 1 :1000
    s = [1, 1 + randperm(100), 102]; % s 为中间 2 到 101 的排列组合
    temp = 0;
    for j = 1 : 101
        temp = temp + d(s(j), s(j + 1));
    end
    if temp < Sum
        Sum = temp;
        S0 = s;         % 保存当前最小值的排序情况
    end
end

e = 0.1 ^ 30;           % 选定退火的终止温度
at = 0.999              % 选定的降温系数  T = at * T;
L = 20000;
T = 1;

%% 退火过程
for k = 1 :L
    c = 2 + floor(100 * rand(1, 2));
    c = sort(c);
    c1 = c(1);c2 = c(2);
    df = d(S0(c1 - 1), S0(c2)) + d(S0(c1), S0(c2 + 1)) ...
        - d(S0(c1 - 1), S0(c1)) - d(S0(c2), S0(c2 + 1));
    if (df < 0)
        S0 = [S0(1 :c1 - 1), S0(c2 : -1: c1), S0(c2 + 1 :102)];
        Sum = Sum + df;
    elseif exp(-df / T) > rand(1) 
        S0 = [S0(1 : c1 - 1), S0(c2 : -1: c1), S0(c2 + 1 :102)];
        Sum = Sum + df;
    end
    T = T * at;
    if T < e
        break;
    end
end

S0, Sum

plot (sj(S0, 1) / pi * 180, sj(S0, 2) / pi * 180);
hold on;
plot (sj(S0, 1) / pi * 180, sj(S0, 2) / pi * 180, 'rx');
axis([-5 75 -5 45]);
```
<!--
> 　　我认识你，我永远记得你．那时候你还很年轻，人人都说你美，现在，我是特来告诉你，对我来说，我觉得现在你比年轻时更美，与你那时的面容相比，我更爱你现在备受摧残的面容
-->