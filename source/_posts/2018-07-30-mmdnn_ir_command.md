---
title: MMdnn 的命令行工具
category: CNN
tags: [CNN, MMdnn]
---

使用 `python` 的包管理工具安装 `mmdnn`

```shell
pip install mmdnn
```

会得到以下命令行工具

- `mmdownload`
  用于下载指定框架已训练好的网络模型
- `mmconvert`
  是 `mmtoir`, `mmtocode`, `mmtomodel` 三者的集成，一行命令完成从一个框架模型到另一个框架模型的转换，分为三步常用于调试
- `mmtoir`
  用于将模型转换为 mmdnn 中间表达形式。使用这个命令会得到三个文件
  - `json` 用于[在线](http://mmdnn.eastasia.cloudapp.azure.com:8080/)可视化
  - `proto|pb` 用于描述网络结构模型
  - `npy` 用于保存网络数值参数
- `mmtocode`
  用于将 `mmtoir` 的 `pb`, `npy` 文件转换为在特定框架下构造该网络的原始代码，以及构建网络过程中用于设置权重的参数。
- `mmtomodel`
  用于将 `mmtocode` 的输出 `py` 文件转换为可以直接被指定框架读取的模型文件
- `mmvismeta`
  目前只发现可以使用 `tensorboard` 作为后端将计算图可视化

---

目前遇到的几个问题

- MMdnn 中间表示中算子可以有 38 种，但是新框架内部却有 54 种算子，经过统计 mmdnn 中只有 23 种可以与新框架中的 25 种算子可以对应(存在一对多，多对多关系)，余下仅凭描述无法找到映射关系，何况新框架内部还有自定义算子。
  经过测试(从一个模型转到 IR 层再转回去)模型文件的大小并没有发生变化(在 mb 级别)，说明 mmdnn 在转换中并没有丢失信息。这应该是表示 38 个算子应该是可以描述完整新框架的 54 个算子的。
  猜测 mmdnn 是用细粒度的算子的组合表达其他框架的特殊算子，观察到 IR 层的可视化表示比原模型的层数多。
- MMdnn 中间表示形式中的权值文件使用 `npy` 格式存储，目前没有找到 C++ 相关的官方 API.
  没办法只好用 Python 重写

---

忽然发现这是第 100 篇，mark 一下

2018-01-25 $\rightarrow$ 2018-08-1