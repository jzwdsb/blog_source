---
layout: post
category: C++
date: 2018-02-02
title: c++ 中 std::string 按值传递和按引用 bug
description: 未找到解决方案 
---

　　c++ 中被调用函数声明如下

```c++
static bool breadth_search(const std::string& node);
```

　　调用方式大致如下

```c++
static int layer_start = 0;
static std::vector<std::string> layer;
breadth_search(layer[layer_start]);
```

　　运行时，在 breadth_search 中设置断点 gdb 查看参数变量 node 出现如下错误

```
<error reading variable: Cannot create a lazy string with address 0x0, and a non-zero length.>{static npos = 0xffffffffffffffff, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x0}, _M_string_length = 0x60d2b0, {_M_local_buf = "\000\000\000\000\000\000\000\000\060\375\312\000\000\000\000", _M_allocated_capacity = 0x0}}
```

　　猜测问题原因仍与上次 debug string 时相同，与 libstdc++ 中 string 实现有关

　　调用方式改为如下后问题消失

```c++
std::string  node = layer[layer_start];
breadth_search(node);
```

　　声明方式改为如下后效果相同

```c++
static void breadth_search(std::string node);
```

　　可见问题是无法创建一个 vector<string> 中的元素的引用或者 Copy on write 共享对象，该字符串对象必须至少复制一次，应该可以禁用 c++11 ABI 避免(未尝试)，最简单的还是将函数声明改为按值传递．