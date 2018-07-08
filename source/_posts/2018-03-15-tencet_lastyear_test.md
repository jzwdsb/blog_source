---
layout: post
category: 杂记
date: 2018-03-15
title: 腾讯 2017 年暑期实习生笔试题目
description: 三道算法题
mathjax: true
---

　　总共三道题目，时间限制在 60 分钟之内，出处是[牛客网](https://www.nowcoder.com/7171303)

>　　给定一个字符串s，你可以从中删除一些字符，使得剩下的串是一个回文串。如何删除才能使得回文串最长呢？<br>　　输出需要删除的字符个数。

　　刚看到这道题时，没什么思路(还是太菜)，简单看了下其他人的提示，说到求反转字符串和原字符串的最长公共子序列长度，这道题就变的很简单了．<br>
　　既然是构造回文串，那么就利用回文串正向和反向相同的特性，从输入的字符串构造出的最长回文串一定是与其反转后的字符串的最长公共子序列，答案是要求需要删除多少个字符，那么这个问题就可以转换为求一个字符串与其反转字符串的最大公共子序列长度．

　　最大公共子序列问题是属于动态规划问题，动态规划问题一般要求出一个问题的递推公式.<br>
　　最大公共子序列问题的递推公式为

$$
C(i, j) =
\begin{cases}
0,& \text{$i = 0$ or $j = 0$} \\\
C(i - 1, j - 1) + 1, & \text{$x_i = y_i$} \\\
max(C(i - 1, j), C(i, j - 1)), & \text{$x_i \neq y_i$}
\end{cases}
$$

　　实现代码

```C++
#include <string>
#include <vector>
#include <algorithm>

using std::vector;
using std::string;

using std::max;

int longest_common_substring(string& a, string& b)
{
    vector<vector<int>> mat(a.length() + 1, vector<int>(b.length() + 1, 0));
    for (int i = 0; i < a.length(); ++ i)
    {
        for (int j = 0; j < b.length(); ++j)
        {
            if (a[i] == b[j])
            {
                mat[i + 1][j + 1] = mat[i][j] + 1;
            }else
            {
                mat[i + 1][j + 1] = max(mat[i][j + 1], mat[i + 1][j]);
            }
        }
    }
    return mat[a.length()][b.length()];
}

int construct_palindromes(string str)
{
    string r_str(str.rbegin(), str.rend());
    return str.length() - longest_common_substring(str, r_str);
}
```

> 小Q最近遇到了一个难题：把一个字符串的大写字母放到字符串的后面，各个字符的相对位置不变，且不能申请额外的空间。 
> 你能帮帮小Q吗？

　　刚看到这个问题时想到了 stl 中的 `std::rotate`, 含义是按照给定的参数在给定区间内循环左移，利用这个库函数实现具体交换的细节，它的时间复杂度为 $O(n)$, $n$ 为给定的区间长度，空间复杂度为 $O(1)$, 符合题目中不能申请额外空间的要求．
　　第一次实现时考虑的并不多，发现大写字母确实交换到了尾部，但是其顺序和预料的不同，之后发现错误在于交换时的区间右端并不应是 `str.end()`, 而是第一个交换到尾部的大写字母，这样改正后输出便是对的．

```C++
#include <string>
#include <algorithm>

using std::string;
using std::rotate;

string& capital_rotate(string& str)
{
    string::iterator rotate_end = str.end();
    for (string::iterator it = str.begin(); it not_eq rotate_end; ++it)
    {
        if (isupper(*it))
        {
            auto next_lower = it;
            while (isupper(*next_lower) and next_lower not_eq rotate_end)
                ++next_lower;
            if (next_lower == rotate_end)
                return str;
            rotate(it, next_lower, str.end());
            --rotate_end;
        }
    }
    return str;
}
```

> 　　小Q今天在上厕所时想到了这个问题：有n个数，两两组成二元组，差最小的有多少对呢？差最大呢？

　　看到这个问题时想起了以前做过的一道题，在一个数组中求和等于给定数的两数的下标，这道题不同在于只需要给出个数即可，所以可以排序，情况细分可以有以下几种.

　　如果数组中所有元素相同，那么差最大个数等于差最小个数，那么结果为: $$差最大个数 = 差最小个数 = n \times (n - 1) / 2$$.

　　统计各个数字出现的次数，如果出现超过 1 次，　那么差最小为 0, 如果不同元素出现多次，那么差最小对数等于不同数字内部两两组合之和．<br>
　　这里的具体实现使用了 `std::unique`, 将所有重复的元素移至数组后方，只需对重复的元素计数即可．<br>
　　如果没有重复数字，那么差最小一定出现在两两相邻之间，遍历一边可得，不同两元素间可能出现相同的最小差，注意个数即可.

　　差最大的对数等于最大值的个数$$\times$$最小值的个数

```C++
#include <vector>
#include <map>

#include <algorithm>

using std::map;
using std::vector;
using std::pair;

using std::sort;
using std::make_pair;

using ans_type = pair<int, int>;

int fact(int n)
{
    if (n == 1 or n == 0)return 1;
    int ans = 1;
    for (int i = 2; i < n + 1; ++i)
    {
        ans *= i;
    }
    return ans;
}

int combin(int n, int m)
{
    return fact(n) / (fact(m) * fact(n - m));
}

ans_type min_subs_max_subs(vector<int>& data)
{
    ans_type ans(0, 0);
    vector<int> subs;
    sort(data.begin(), data.end());
    int sub_max;
    int front_count = 0, back_count = 0;
    front_count = static_cast<int>(std::count(data.begin(), data.end(), data.front()));
    back_count = static_cast<int>(std::count (data.begin(), data.end(), data.back()));
    ans.second = front_count * back_count;
    
    auto min_pos = std::min_element(subs.begin(), subs.end());
    int min_count = static_cast<int>(std::count(subs.begin(), subs.end(), *min_pos));
    auto new_end = std::unique(data.begin(), data.end());
    map<int, int> unique_count;
    if (new_end == data.end())
    {
        ans.first = min_count;
    }else
    {
        for (auto it = new_end; it not_eq data.end(); ++it)
        {
            if (unique_count.count(*it) == 0)
            {
                unique_count.insert(make_pair(*it, 2));
            }else
            {
                ++unique_count[*it];
            }
        }
        ans.first = 1;
        for (auto &it : unique_count)
        {
            ans.first += combin(it.second, 2);
        }
    }

    return ans;
}
```

　　以上，感觉来说并不难．

<!--
> 　　他真诚地错把自己的肉欲当作浪漫的恋情，错把自己的优柔寡断视为艺术家的气质，还错把自己的无所事事看成哲人的超然物外．他心智平庸，却孜孜追求高尚娴雅，因而从他眼睛望去，所有的事物都蒙上一层感伤的金色雾纱，轮廓模糊不清，结果就显得比实际的形象大些．
-->
