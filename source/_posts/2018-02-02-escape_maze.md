---
layout: post
category: algorithms
date: 2018-02-02
title: 模拟逃出迷宫
description: 使用去重优化
---

```
UDDLUULRUL
UURLLLRRRU
RRUURLDLRD
RUDDDDUUUU
URUDLLRRUU
DURLRLDLRL
ULLURLLRDU
RDLULLRDDD
UUDDUDUDLL
ULRDLUURRR
```

>  　100 * 100 矩阵中，每个元素为 U, D, L, R 四个字符分别代表走到此格时要转向的方向，
> 开始时有 100 人均匀分布在此矩阵中，问最后能走出矩阵而不在里面兜圈子的人数

## 分析

　　直观的就是对每个格子进行模拟，直到走出或者回到之前路过的小格，但是这样将有大量的重复运算，
故另建一个矩阵表示每个点的状态，分为四种

1. NOPASSBY
2. UNKNOWN
3. FAILURE
4. SUCCESS

　　四种状态分别表示未曾处理过，未知，失败，成功．在模拟过程中缓存经过的路径，当模拟的坐标超出边界或者遇到已经设为 SUCCESS 的点后
将整条路的状态设置为 SUCCESS, 并将结果计数加上整条路的长度；当遇到 FAILURE 或者 UNKNOWN 时，将整条路设置为 FAILURE. 模拟过程中遇到的点属于未知状态，
设置为 UNKNOWN.

## 完整代码

```c++
/** 计算过程中的三种状态
 *  0 未知
 *  1 未知
 *  2 失败
 *  3 成功
 * */

#define NOPASSBY    0
#define UNKNOWN     1
#define FAILURE     2
#define SUCCESS     3


struct coordinate
{
    int x = 0;
    int y = 0;
    coordinate(int x, int y):x(x), y(y){}
};

static int escape;
static vector<vector<int>> point;

static void set_status(vector<coordinate>& path, int status)
{
    for (auto &it : path)
    {
        point[it.x][it.y] = status;
    }
}

static void find_path(vector<string>& map, int x, int y)
{
    vector<coordinate> path;
    do
    {
        point[x][y] = UNKNOWN;
        path.emplace_back(x, y);
        switch (map[x][y])
        {
            case 'U': --x; break;
            case 'D': ++x; break;
            case 'L': --y; break;
            case 'R': ++y; break;
            default: break;
        }
    }while (x < map.size() and y < map[0].length() and x >= 0 and y >= 0 and point[x][y] == NOPASSBY);
    if (x >= map.size() or y >= map[0].length() or x < 0 or y < 0 or point[x][y] == SUCCESS)
    {
        escape += path.size();
        set_status(path, SUCCESS);
        return;
    }
    if (point[x][y] == UNKNOWN or point[x][y] == FAILURE)
    {
        point[x][y] = FAILURE;
        set_status(path, FAILURE);
        return ;
    }
}


int count_escape(vector<string>& map)
{
    escape = 0;
    point.resize(map.size(), vector<int>(map[0].length(), NOPASSBY));
    for (int i = 0; i < map.size(); ++i)
    {
        for (int j = 0; j < map[0].length(); ++j)
        {
            if (point[i][j] == NOPASSBY)
            {
                find_path(map, i, j);
            }
        }
    }
    return escape;
}
```