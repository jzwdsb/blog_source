---
title: Python 调用 C/C++ 动态库
description: ctypes 模块的简单使用
category: Python
date: 2018-08-07
tags: [C/C++, Python]
---

Python 调用 C/C++ 动态库可以使用 `ctypes` 模块提供的功能。
需要注意的是在使用 C++ 时对导出符号要使用 `extern "C"` 声明，C++ 编译器会对符号名做名字重整(name mangling), 使导出符号不被 `ctypes` 模块识别，`extern "C"` 声明要求编译器对符号按 C 语言的规则处理。`ctypes` 只能识别 `C` 导出的符号。

# C/C++

```C++
static int foo_(int)
{
    //do something
    return i;
}

extern "C"{
    int foo(int i)
    {
        return foo_(i)
    }
}
```

# 编译

`gcc/g++` 编译选项包括 `-shared` 选项

```shell
g++ -shared -fPIC foo.cpp -o foo.so
```

- `-shared`
  编译为动态链接库
- `-fPIC`
  生成位置无关代码，这样链接库中的地址全为相对地址，为解决载入链接库时的地址重定位问题。

# Python

使用 `ctypes` 模块。

```Python3
from ctypes import *

foo = cdll.LoadLibrary('foo.so').foo
pam = c_int(1)
print (foo(pam.value))
```