---
category: algorithm
date: 2018-01-30
title: 逆序数
mathjax: true
---

> 在数列 $ A=\{a_0, a_1, ..., a_{n-1} \} $中，如果一组数(i, j)满足$a_i > a_j$ 且 i < j,
>那么这组数就称为一个逆序，数列A中的逆序的数量称为逆序数．逆序数与在该数列上执行冒泡排序交换的次数相等．
> 求给定数列A的逆序数，如果直接使用冒泡排序会导致 Time Limit Exceeded.

# 分析

　　由于采用冒泡排序计算逆序数在大数组上会导致超时，无法在限定时间内算出正确结果，故采用分治策略，在分支策略中，
选择使用归并排序．

　　根据分治策略，将数列划分为左右两个子数列 L, R 后，该数列的逆序数 = L的逆序数 + R的逆序数 + 合并时两数组时产生的逆序数，由上可得如下代码

```cpp
long count_inversions(vector<int>& data, int low, int high)
{
    long v1, v2, v3;
    int mid = (low + high) / 2
    if (low + 1 < high)
    {
        v1 = count_inversions(data, low, mid);
        v2 = count_inversions(data, mid, high);
        v3 = merge_sort(data, low, mid, high);
        return v1 + v2 + v3;
    }
    return 0;
}
```

```cpp
llong count_inversions(vector<int>& data, int low, int high)
{
    llong v1, v2, v3;
    int mid = (low + high) / 2
    if (low + 1 < high)
    {
        v1 = count_inversions(data, low, mid);
        v2 = count_inversions(data, mid, high);
        v3 = merge_sort(data, low, mid, high);
        return v1 + v2 + v3;
    }
    return 0;
}
```

　　两数列合并时的逆序数应该等于 L 中的元素移动到 R 的各个元素之后的数量之和，R 每个元素单独计算．
设L长度为 n1, L 当前下标为 i , R 当前下标为 j ，只要在合并R各元素时计算 n1 - i, 就能知道L中多少个元素移动到 R[j] 后方，
即当前有多少个与 R[j] 相关的逆序数．由上可得如下代码

```Cpp
llong merge_sort(vector<int>& a, int low, int mid, int high)
{
    int i = 0, j = 0, k;
    llong cnt = 0;
    vector<int> left(a.begin() + low, a.begin() + mid);
    vector<int> right(a.begin() + mid, a.begin() + high);
    for (k = low; k < high and i < left.size() and j < right.size(); ++k)
    {
        if (left[i] <= right[j])
        {
            a[k] = left[i];
            ++i;
        }else
        {
            a[k] = right[j];
            ++j;
            cnt += left.size() - i;
        }
    }
    while (i < left.size())
    {
        a[k] = left[i];
        ++k;
        ++i;
    }
    while (j < right.size())
    {
        a[k] = right[j];
        ++k;
        ++j;
    }
    return cnt;
}
```

# 完整代码

```Cpp
llong merge_sort(vector<int>& a, int low, int mid, int high)
{
    int i = 0, j = 0, k;
    llong cnt = 0;
    vector<int> left(a.begin() + low, a.begin() + mid);
    vector<int> right(a.begin() + mid, a.begin() + high);
    for (k = low; k < high and i < left.size() and j < right.size(); ++k)
    {
        if (left[i] <= right[j])
        {
            a[k] = left[i];
            ++i;
        }else
        {
            a[k] = right[j];
            ++j;
            cnt += left.size() - i;
        }
    }
    while (i < left.size())
    {
        a[k] = left[i];
        ++k;
        ++i;
    }
    while (j < right.size())
    {
        a[k] = right[j];
        ++k;
        ++j;
    }
    return cnt;
}

llong count_inversions(vector<int>& data, int low, int high)
{
    llong v1, v2, v3;
    int mid = (low + high) / 2;
    if (low + 1 < high)
    {
        v1 = count_inversions(data, low, mid);
        v2 = count_inversions(data, mid, high);
        v3 = merge_sort(data, low, mid, high);
        return v1 + v2 + v3;
    }
    return 0;
}
```

<!--
>　　我知道我只能陪你走这一段路，再见之后，各自安静生活数年．然后在某个人潮拥挤的街头，
>透过公车的玻璃突然看见你．我想叫司机马上停车，想用力拍打窗户吸引你的注意，想从车上跳下来，
>想奔跑，想大喊大叫，想把整个阻隔在你我之间的世界撕裂．我呼吸急促，面额潮红，手指颤抖．
>我在激烈的想像中把自己感动的快哭了．而事实总是，我一动不动地的坐着，安静的看你远去．
>你的脸，从开始到现在，我原来从未看清楚过．
-->