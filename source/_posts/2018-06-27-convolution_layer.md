---
title: NCNN 源码剖析(6) Convolution Layer
description: Convolution Layer 层
date: 2018-06-27 23:00
category: CNN
tags: [CNN, ncnn]
mathjax: true
---

卷积层(Convolution Layer)的声明在 `./src/layer/convolution.h` 中，继承了 `Layer` 并重写了其 `load_param`, `load_model` 和 `forward` 接口.

## 接口声明

### 公有接口

```C++
virtual int load_param(const ParamDict& pd);
virtual int load_model(const ModelBin& mb);
virtual int forward(const Mat& bottom_blob, Mat& top_blob) const;
```

在以上接口中

- 载入网络参数和模型
  - `load_param`
  - `load_model`
- 前向传播接口
  - `forward`

### 成员属性

```C++
// param
int num_output;
int kernel_w;
int kernel_h;
int dilation_w;
int dilation_h;
int stride_w;
int stride_h;
int pad_w;
int pad_h;
int bias_term;

int weight_data_size;

// model
Mat weight_data;
Mat bias_data;
```

- 参数信息
  - 输出个数
    - `num_output`
  - 卷积核的大小
    - `kernel_w`
    - `kernel_w`
  - 伸缩尺度
    - `dilation_w`
    - `dilation_h`
  - 步长
    - `stride_w`
    - `stride_h`
  - 填充大小
    - `pad_w`
    - `pad_h`
  - 偏置项
    - `bias_term`
  - 权重数据大小
    - `weight_data_size`
- 模型数据
  - 权重数据
    - `weight-data`
  - 偏置数据
    - `bias_data`

## `fordward` 前向传播接口实现

- 如果输入 `blob` 和卷积核大小为 $1 \times 1$ 为一维向量，则使用新创建內积层并将计算交给內积层，之后返回计算结果
- 根据输入参数将输入 `blob`  `border` 扩展
- 创建一个和卷积核大小相同的矩阵用来表示之后卷积运算时与卷积核 `element wise` 相乘求和的输入 `blob` 中的元素的偏移量。这样能够方便计算。
- 计算的同时将结果输出在 `top_blob` 中。

## 源码

### 接口声明源码

```C++
class Convolution : public Layer
{
public:
    Convolution();

    virtual int load_param(const ParamDict& pd);

    virtual int load_model(const ModelBin& mb);

    virtual int forward(const Mat& bottom_blob, Mat& top_blob) const;

public:
    // param
    int num_output;
    int kernel_w;
    int kernel_h;
    int dilation_w;
    int dilation_h;
    int stride_w;
    int stride_h;
    int pad_w;
    int pad_h;
    int bias_term;

    int weight_data_size;

    // model
    Mat weight_data;
    Mat bias_data;
};
```

### 接口实现源码

重点在于前向传播实现

```C++
int Convolution::forward(const Mat& bottom_blob, Mat& top_blob) const
{
    // convolv with NxN kernel
    // value = value + bias

    // flattened blob, implement as InnerProduct
    if (bottom_blob.dims == 1 && kernel_w == 1 && kernel_h == 1)
    {
        int num_input = weight_data_size / num_output;
        if (bottom_blob.w == num_input)
        {
            // call InnerProduct
            ncnn::Layer* op = ncnn::create_layer(ncnn::LayerType::InnerProduct);

            // set param
            ncnn::ParamDict pd;
            pd.set(0, num_output);
            pd.set(1, bias_term);
            pd.set(2, weight_data_size);

            op->load_param(pd);

            // set weights
            ncnn::Mat weights[2];
            weights[0] = weight_data;
            weights[1] = bias_data;

            op->load_model(ModelBinFromMatArray(weights));

            // forward
            op->forward(bottom_blob, top_blob);

            delete op;

            return 0;
        }
    }

    int w = bottom_blob.w;
    int h = bottom_blob.h;
    int channels = bottom_blob.c;

//     fprintf(stderr, "Convolution input %d x %d  pad = %d %d  ksize=%d %d  stride=%d %d\n", w, h, pad_w, pad_h, kernel_w, kernel_h, stride_w, stride_h);

    const int kernel_extent_w = dilation_w * (kernel_w - 1) + 1;
    const int kernel_extent_h = dilation_h * (kernel_h - 1) + 1;

    Mat bottom_blob_bordered = bottom_blob;
    if (pad_w > 0 || pad_h > 0)
    {
        copy_make_border(bottom_blob, bottom_blob_bordered, pad_h, pad_h, pad_w, pad_w, BORDER_CONSTANT, 0.f);
        if (bottom_blob_bordered.empty())
            return -100;

        w = bottom_blob_bordered.w;
        h = bottom_blob_bordered.h;
    }
    else if (pad_w == -233 && pad_h == -233)
    {
        int wpad = kernel_extent_w + (w - 1) / stride_w * stride_w - w;
        int hpad = kernel_extent_h + (h - 1) / stride_h * stride_h - h;
        if (wpad > 0 || hpad > 0)
        {
            copy_make_border(bottom_blob, bottom_blob_bordered, hpad / 2, hpad - hpad / 2, wpad / 2, wpad - wpad / 2, BORDER_CONSTANT, 0.f);
            if (bottom_blob_bordered.empty())
                return -100;
        }

        w = bottom_blob_bordered.w;
        h = bottom_blob_bordered.h;
    }

    int outw = (w - kernel_extent_w) / stride_w + 1;
    int outh = (h - kernel_extent_h) / stride_h + 1;

    top_blob.create(outw, outh, num_output);
    if (top_blob.empty())
        return -100;

    const int maxk = kernel_w * kernel_h;

    // kernel offsets
    std::vector<int> _space_ofs(maxk);
    int* space_ofs = &_space_ofs[0];
    {
        int p1 = 0;
        int p2 = 0;
        int gap = w * dilation_h - kernel_w * dilation_w;
        for (int i = 0; i < kernel_h; i++)
        {
            for (int j = 0; j < kernel_w; j++)
            {
                space_ofs[p1] = p2;
                p1++;
                p2 += dilation_w;
            }
            p2 += gap;
        }
    }

    // num_output
    #pragma omp parallel for
    for (int p=0; p<num_output; p++)
    {
        float* outptr = top_blob.channel(p);

        for (int i = 0; i < outh; i++)
        {
            for (int j = 0; j < outw; j++)
            {
                float sum = 0.f;

                if (bias_term)
                    sum = bias_data[p];

                const float* kptr = (const float*)weight_data + maxk * channels * p;

                // channels
                for (int q=0; q<channels; q++)
                {
                    const Mat m = bottom_blob_bordered.channel(q);
                    const float* sptr = m.row(i*stride_h) + j*stride_w;

                    for (int k = 0; k < maxk; k++) // 29.23
                    {
                        float val = sptr[ space_ofs[k] ]; // 20.72
                        float w = kptr[k];
                        sum += val * w; // 41.45
                    }

                    kptr += maxk;
                }

                outptr[j] = sum;
            }

            outptr += outw;
        }
    }

    return 0;
}
```