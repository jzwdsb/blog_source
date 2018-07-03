---
title: 矩阵快速幂
date: 2018-05-22 18:53:42
category: 算法
mathjax: true
description: 原理和实现
---

## 原理

　　矩阵快速幂算法可以用在斐波那契数列的计算中，公式如下
　　斐波那契数列定义
$$
    a_n = a_{n - 1} + a_{n - 2}, a_0 = 1, a_1 = 1
$$
　　矩阵形式的斐波那契公式形式为
$$
\begin{bmatrix}
F_{n + 1} & F_{n} \\\\
F_n & F_{n - 1}
\end{bmatrix}
= 
{
\begin{bmatrix}
1 & 1 \\\\
1 & 0 \\\\
\end{bmatrix}
}^n
$$
　　虽然矩阵形式的递推公式本质上与原公式并无不同，但是对于矩阵运算可以有特定的加速方法，比如这里看出 $F_n$ 可以从一个矩阵的 n 次幂得到，所以这里可以使用矩阵快速幂算法.
　　矩阵快速幂的实现与普通实数快速幂的实现并无本质上不同，都是二分法的思想，这里不再赘述实数快速幂的实现.

## 实现

　　一次矩阵相乘的时间复杂度为$O(n^3)$, 这里可以做缓存友好的优化，所以矩阵快速幂的时间复杂度为$O(m^3\log_2{n}), \text{m 为矩阵大小}$.

```C++
using matrix = std::vector<std::vector<int>>;
/** 矩阵相乘底层算法*/
matrix operator *(const matrix& a, const matrix& b)
{
    assert(a[0].size() == b.size());
    matrix ans(b.size(), std::vector<int>(a[0].size(), 0));
    for (int i = 0;  i < a[0].size(); ++i)
    {
        for (int j = 0; j < b.size(); ++j)
        {
            for (int k = 0; k < b[0].size(); ++k)
            {
                ans[i][j] += a[i][k] * b[k][j];
            }
        }
    }
    return ans;
}

/** 递归形式*/
matrix fast_matrix_power(matrix mat, int n)
{
    if (n == 1)
        return mat;
    matrix ans = fast_matrix_power(mat, n / 2);
    ans = ans * ans;
    return n % 2 == 0 ? ans : ans * mat;
}

/** 迭代形式*/
matrix fast_matrix_power_it(matrix mat, int n)
{
    matrix ans = mat;
    while (n not_eq 0)
    {
        ans = ans * ans;
        if (n % 2 == 1)
        {
            ans = ans * mat;
        }
        n /= 2;
    }
    return ans;
}
```


