---
layout: post
category: algorithm
date: 2018-02-02
title: 一个图论问题
description: 广搜和去重
tags: [algorithm]
---

>　　如图所示，９个盘子排成圆圈，八个盘子中有蚱蜢，一个空盘，每只蚱蜢可以跳到相邻的空盘中，
> 或者隔一只蚱蜢跳到空盘上，从当前状态到 1-8 换位, 2-7 换位....至少经过多少次跳跃

![insect_jump.png](/downloads/insect_jump.png)

## 分析

　　这个问题可以转化为图论问题，以图所示状态为起始状态，通过广搜，求出到目标状态的步长．

　　状态的表示，由于图上为一个环形，所以用字符串表示状态，字符串索引时表示为环形数组．

```c++
#define INDEX(i) (((i) + 9) % 9)
```

　　状态转移时将空盘与左右相邻四个字符每个交换一次产生新状态，判断是否到目标状态，将每个状态加入到一个 set 中去重，每次深入一层便将 steps 加一，最后当到达目标状态后，返回 steps 计数

## 完整代码

```c++
#include <string>
#include <vector>
#include <unordered_set>

/** 实现循环数组*/
#define INDEX(i) (((i) + 9) % 9)


static int steps = 0;
/** 图的点集，每产生一个状态就将当前状态点加入到点集中*/
static std::unordered_set<std::string> graph;
static std::string start;


static std::queue<std::string> layer;
static int layer_start = 0;

/** 这个函数产生当前状态经一步转移可到达的所有状态并插入到 layer 之后*/
static bool breadth_search(const std::string& node)
{
    int empty_pos = static_cast<int>(node.find('0'));
    for(int i = empty_pos - 2; i < empty_pos + 2; ++i)
    {
        std::string child = node;
        std::swap(child[INDEX(i)], child[INDEX(empty_pos)]);
        if (child == "087654321")
        {
            return true;
        }
        /** 避免重复搜索判断当前状态是否已经存在于图中*/
        if (graph.count(child) == 0)
        {
            layer.push_back(child);
            graph.insert(child);
        }
    }
    return false;
}

int insect_jump()
{
    start = "012345678";
    layer.push_back(start);
    graph.insert(start);
    layer_count = 0;
    step = 0;
    /** 当到达目的状态时，搜索结束*/
    while (true)
    {
        /** 当前层搜索完毕，进入下一层*/
        if (layer_count == 0)
        {
            ++steps;
            layer_count = static_cast<int>(layer.size());
        }
        std::string node = layer.front();layer.pop();
        if (breadth_search(node))
        {
            break;
        }
        --layer_count;
    }
    return steps;
}
```

<!--
> 　　我以前是有些得意，自信，更多的是不在乎．那些不在乎里，含着一部分自知得不到的不屑，很幼稚，还觉得浪漫，想着即使最冷的天气也还是一件大衣可以出门，在餐厅里看奔波的路人，然后想着他们都是要回家的，而我会是一个赶路人．但是这些在遇见一个人后都会改变，开始按部就班地攒钱看房，顾及户口，职位，她让你的生活的更落地，也更谦卑，你开始觉得世界很大，生活也很神圣，细着心去经营也害怕出错，再也不敢对任何美景吹口哨，看见灰尘也不觉得陈旧．前几天做了一个梦，梦见我回到了长春，冬天，在桂林路的麦当劳等人，靠窗的座位，外面下着雪，人们披着一身的雪债匆匆赶路，他们聊天时有哈气，像真正的寒暄．我就这样看了很久，等的人一直没来，我也没打算走，醒来后我靠在床头上，像个老人一样沮丧了很久
-->