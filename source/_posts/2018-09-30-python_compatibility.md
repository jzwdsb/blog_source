---
title: python 2-3 的兼容代码的编写
date: 2018-09-30
category: Python
tags: [Python]
description: six 模块的使用及其他注意事项
---

目前需要编写 python2-3 兼容的代码，并且对之前的 python2 的代码做 python3 的兼容(~~说顺便把这个做了, 几千行的代码说顺便就顺便~~))，需要注意些细节。

## python 2-3 的不兼容

python3 与 python2 的不兼容的地方主要有以下几点

- `print` 由关键字变为函数
- `raw_input` 在 Python3 中变为 `input`
- `urllib` 重新组织
- `xrange` 在 Python3 中重命名为 `range`
- `raise` 语法改变
- `except` 语法改变
- python3 支持 `exception chain`
- `/` python3 变为浮点计算
- 类型的重命名
- `utf8` 的支持
- `dict` 的 iterate 方法重命名

## 注意事项

使用 `six` 模块屏蔽 Python2 和 Python3 的差异部分。
`six` 模块的详细文档见[官网](https://pythonhosted.org/six/)

使用 `six.PY2`, `six.PY3` 检查当前是运行在 `Python2` 还是 `Python3` 环境下。

## 常用措施

- `six.print_` 代替 `print`
- `six.move` 代替 `python3` 中重命名或重新组织的模块, 如 `xrange`
- `six.iteritems` 代替 `dict` 等容器的迭代
- `requests` 代替 `urllib`

目前较为常用的就这些，待续