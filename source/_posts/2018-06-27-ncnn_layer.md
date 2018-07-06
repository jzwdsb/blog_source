---
title: NCNN 源码剖析(三) Layer
description: NCNN 中的 Layer
date: 2018-06-27 15:00
category: CNN
tags: [CNN, ncnn]
---

`Layer` 类在头文件中声明, 源码见末尾.

这里 `Layer` 是作为输入层，卷积层，池化层,softmax层等的基类出现的。这类类作为神经网络中的计算单元。继承 `Layer` 层的类都在 `./src/layer/` 下面. <!--整理到另一个文件中，见此[链接](layer_class_list). -->

## 接口声明

### 公开接口

`Layer` 公开的接口有

```C++
virtual int load_param(const ParamDict& pd);
virtual int load_model(const ModelBin& mb);
virtual int forward(const std::vector<Mat>& bottom_blobs, std::vector<Mat>& top_blobs)const;
virtual int forward(const Mat& bottom_blobs, Mat& top_blobs);
virtual int forward_inplace(const std::vector<Mat>& bottom_blob, std::vector<Mat>& top_blob);
virtual int forward_inplace(const Mat& bottom_blob, Mat& top_blob);
```

这些接口都要其子类重写，以上接口主要分为两类

- 载入参数和数据
  - `virtual int load_param`
  - `virtual int load_model`
- 前向传播
  - `virtual int forward`
  - `virtual int forward_inplace`

### 公有属性

其公有属性如下

```C++
bool one_blob_only;
bool support_inplace;
#if NCNN_STRING
std::string type;
std::string name;
#endif
std::vector<int> bottoms;
std::vector<int> tops;
```

每个成员都有注释解释它的功能(~~所以我的很大一部分工作就只是翻译而已~~)

- `one_blob_only`
    如果为真，那么这一层只有一个输入 `blob` 和一个输出 `blob`
- `support_inplace`
    是否支持就地计算
- `type`
    在 `NCNN_STRING` 宏开启时，支持使用字符串指定 `type` (层的类别)
- `name`
    在 `NCNN_STRING` 宏开启时，支持使用字符串制定 `name` (层的名字)
- `bottoms`
    作为这一层输入的 `blob` 的 index
- `tops`
    作为这一层输出的 `blob` 的 index

## Layer 实现源码分析

### `forward` 接口

`Layer` 中 `forward` 接口只是要求其子类重写这个接口实现前向传播的计算

接口声明如下

```C++
int forward(const Mat& bottom_blob, Mat& top_blob);
```

这一层的 `Layer` 以前一层的 `bottom_blob` 作为输入，`top_blob` 为输出参数，为后一层的输出参数.

`Mat` 可以理解为前一层网络计算出的 `feature map`, 负责在网络中传递和存储数据.

## 源码

### 接口声明源码

```C++
class Layer
{
public:
    // empty
    Layer();
    // virtual destructor
    virtual ~Layer();

    // load layer specific parameter from parsed dict
    // return 0 if success
    virtual int load_param(const ParamDict& pd);

    // load layer specific weight data from model binary
    // return 0 if success
    virtual int load_model(const ModelBin& mb);

public:
    // one input and one output blob
    bool one_blob_only;

    // support inplace inference
    bool support_inplace;

public:
    // implement inference
    // return 0 if success
    virtual int forward(const std::vector<Mat>& bottom_blobs, std::vector<Mat>& top_blobs) const;
    virtual int forward(const Mat& bottom_blob, Mat& top_blob) const;

    // implement inplace inference
    // return 0 if success
    virtual int forward_inplace(std::vector<Mat>& bottom_top_blobs) const;
    virtual int forward_inplace(Mat& bottom_top_blob) const;

public:
#if NCNN_STRING
    // layer type name
    std::string type;
    // layer name
    std::string name;
#endif // NCNN_STRING
    // blob index which this layer needs as input
    std::vector<int> bottoms;
    // blob index which this layer produces as output
    std::vector<int> tops;
};

// layer factory function
typedef Layer* (*layer_creator_func)();

struct layer_registry_entry
{
#if NCNN_STRING
    // layer type name
    const char* name;
#endif // NCNN_STRING
    // layer factory entry
    layer_creator_func creator;
};

#if NCNN_STRING
// get layer type from type name
int layer_to_index(const char* type);
// create layer from type name
Layer* create_layer(const char* type);
#endif // NCNN_STRING
// create layer from layer type
Layer* create_layer(int index);

#define DEFINE_LAYER_CREATOR(name) \
    ::ncnn::Layer* name##_layer_creator() { return new name; }
```