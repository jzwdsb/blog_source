---
title: MMdnn IR 层的表示方式
date: 2018-07-27
category: CNN
tags: [CNN, MMdnn]
---

[MMdnn](https://github.com/Microsoft/MMdnn)能够实现在不同的框架之间互相转换网络模型。
其原理是将所有的模型先转化为中间表达形式(Intermediate Representation), 分析下 IR 层的表示方式。

## proto 说明文件

IR 层的 proto 说明文件可见 [github](https://github.com/Microsoft/MMdnn/blob/master/mmdnn/conversion/common/IR/graph.proto)。

其中包含以下几个部分

- GraphDef
  - NodeDef
  - version
- NodeDef
  - name
  - op
  - input
  - attr
- Attrvalue
  - list
  - type
  - shape
  - tensor
- Tensorshape
  - dim
- LiteralTensor
  - type
  - tensor_shape
  - values

## Graph

描述网络模型的 `graph`, 内部包含若干 `node`, 这里的结点神经网络中的层，`node` 中的 `input` 即描述每一层之间的输入输出连接关系.

## NodeDef

- `name`
  本层 `name`, 在 `Graph` 中唯一
- `op`
  本层算子，算子描述见[链接](https://github.com/Microsoft/MMdnn/blob/master/mmdnn/conversion/common/IR/ops.pbtxt),在算子描述文件中，算子 `name` 唯一.
- `input`
  用于描述本层输入关系，各层之间的输入输出关系靠此成员描述
- `attr`
   `attr` 成员可以是 `listvalue`, `type`, `shape`, `tensor`
  - `list`
    存储 list 成员，如 list(int), list(float), list(shape) 等
    - data
      保存数值数据，类型可以是 `bytes`, `int64`, `float`, `bool`
    - `type`
    - `shape`
    - `tensor`  
  - `type`
    描述数值类型
  - `shape`
    描述 tensor 形状
  - `tensor`
    各 `node` 之间传递的 `tensor`

## TensorShape

描述 `tensor` 的维度信息

## LiteralTensor

存储 `tensor`, 在各 `node` 之间传递