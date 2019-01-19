---
layout: post
title: 科赫曲线
date: 2018-01-30
description: 简单的C++语言实现
category: algorithm
tags: [algorithm]
---

> 　　请编写一个程序，输入整数n，输出科赫曲线的定点坐标，该科赫曲线由深度为n的递归调用画出．
> 假设从端点(0, 0)开始，到端点(100, 0)结束，沿连续线段顺次输出定点坐标

　　科赫曲线通过下面的递归操作画出
1. 将线段(p1, p2)三等分
2. 以三等分点s, t为定点作出正三角形(s, u, t)
3. 对线段(p1, s), (s, u), (u, t), (t,. p2)递归重复进行上述操作

# 解法

　　首先根据所给线段算出三等分点

```C++
    s.x = (2 * p1.x + p2.x) / 3.0;
    s.y = (2 * p1.y + p2.y) / 3.0;

    t.x = (p1.x + 2 * p2.x) / 3;
    t.y = (p1.y + 2 * p2.y) / 3;
```

　　以s, y为底边，算出其位于上方的等边三角形第三个顶点的坐标．问题可转化为将一点以另一点
为圆心逆时针(根据点的相对位置全部为逆时针)旋转${60}^\circ$的新的点坐标，经过推导，公式为

$$ x_1 = x \cdot cos\theta - y \cdot sin\theta $$

$$ y_1 = y \cdot cos\theta + x \cdot sin\theta $$

可得如下代码

```C++
    u.x = (t.x - s.x) * cos(th) - (t.y - s.y) * sin(th) + s.x;
    u.y = (t.x - s.x) * sin(th) + (t.y - s.y) * cos(th) + s.y;
```

# 完整代码

```C++
struct point
{
    double x = 0.0;
    double y = 0.0;
};

void koch(int depth, point p1, point p2)
{
    if (depth == 0)
        return ;
    point s, t, u;

    const double th = M_PI * 60.0 / 180.0;

    s.x = (2 * p1.x + p2.x) / 3.0;
    s.y = (2 * p1.y + p2.y) / 3.0;

    t.x = (p1.x + 2 * p2.x) / 3;
    t.y = (p1.y + 2 * p2.y) / 3;

    u.x = (t.x - s.x) * cos(th) - (t.y - s.y) * sin(th) + s.x;
    u.y = (t.x - s.x) * sin(th) + (t.y - s.y) * cos(th) + s.y;

    koch(depth - 1, p1, s);
    printf("%.8f %.8f\n", s.x, s.y);
    koch(depth - 1, s, u);
    printf("%.8f %.8f\n", u.x, u.y);
    koch(depth - 1, u, t);
    printf("%.8f %.8f\n", t.x, t.y);
    koch(depth - 1, t, p2);
}
```

<!--
>　　　　我走到人生的十字路口，总知道那条路是对的，毫无例外，我总是知道，但我从来不走，
>因为太苦了
-->