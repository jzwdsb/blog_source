---
title: 合并 prototxt 和 bin 文件工具实现
category: CNN
date: 2018-07-25
tags: [CNN, caffe]
---

接下来几周的任务是实现 MMdnn 中 IR 层到新框架网络模型的后端转换，mentor 先给了一个合并 prototxt 和 bin 的合并工具源码做参考(感觉不是一个层次上的东西啊)，分析下这个工具的实现

# 网络描述文件

## `prototxt` 网络结构描述文件

`prototxt` 文件形式为纯文本格式，可以直接用常用的文本编辑器打开(vscode, sublime text 等)，caffe 用该类型文件保存并描述网络结构信息，所有内容明文保存。其存储格式并不适合~~一般人~~阅读，以下是 `prototxt` 文件描述 `input` 层和第一层 `conv` 层的格式

```protobuf
layer {
  name: "data"
  type: "Input"
  top: "data"
  input_param { shape: { dim: 1 dim: 3 dim: 227 dim: 227 } }
}
layer {
  name: "conv1"
  type: "Convolution"
  bottom: "data"
  top: "conv1"
  convolution_param {
    num_output: 64
    kernel_size: 3
    stride: 2
  }
}
```

可以通过这个[网站](http://ethereon.github.io/netscope/#/editor)看到图形化的网络结构。

## `bin` 网络参数描述文件

`bin` 文件为二进制文件，无法直接阅读，caffe 使用该类型文件描述网络参数，如卷积层的卷积核的 weight，bias 等数值信息

# 源码分析

经分析，合并大致流程如下

- 读入 `prototxt` 文件
- 读入 `bin` 文件
  - 以上类型为 caffe 中的 `caffe::NetParameter` 类型，存储其网络信息，包含各层结构和数据
- 如果支持输入域
  - 根据 `protobuf` 中描述的输入信息构造输入层 NCHW 信息或者多维结构信息。
- 对 `protobuf` 中描述的每一层 `layer` 做如下操作
  - 根据 `layer.name` 找到对应 `bin` 文件中对应的 `layer` 参数信息
    - 若不存在，继续下一层 `layer`
  - 对找到的 `layer` 参数信息中的每一个 `blob` 做如下操作
    - 获取其 `NCHW` 或者多维结构信息，并记录
    - 将其维度信息和数据合并到单个 `blob` 中并添加到 `layer` 中
- 将合并后的 `NetParameter` 序列化并存入到 `output_model` 中