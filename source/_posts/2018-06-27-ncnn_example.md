---
title: NCNN 源码剖析(一) Example
description: NCNN 中的 example
date: 2018-06-27 11:00
category: CNN
tags: [CNN, ncnn]
mathjax: true
---

## example 解读

ncnn 目录结构下的 `example` 中有一份使用 `squeezenet` 做图像分类的实例程序。

`squeezenet` 是一个轻量级的卷积神经网络，详细知识可见于[此论文](https://arxiv.org/pdf/1602.07360.pdf)

其中关键代码如下

```C++
static int detect_squeezenet(const cv::Mat& bgr, std::vector<float>& cls_scores)
{
    ncnn::Net squeezenet;
    squeezenet.load_param("squeezenet_v1.1.param");
    squeezenet.load_model("squeezenet_v1.1.bin");

    ncnn::Mat in = ncnn::Mat::from_pixels_resize(bgr.data, ncnn::Mat::PIXEL_BGR, bgr.cols, bgr.rows, 227, 227);

    const float mean_vals[3] = {104.f, 117.f, 123.f};
    in.substract_mean_normalize(mean_vals, 0);

    ncnn::Extractor ex = squeezenet.create_extractor();
    ex.set_light_mode(true);

    ex.input("data", in);

    ncnn::Mat out;
    ex.extract("prob", out);

    cls_scores.resize(out.w);
    for (int j=0; j<out.w; j++)
    {
        cls_scores[j] = out[j];
    }

    return 0;
}
```

分析下这份代码的结构

首先创建网络对象 `squeezenet`，读入参数和模型，读取输入数据并调整其尺寸至需要的大小. 使用减去平均数的方法正则化输入数据(Normalization by Subtracting the Mean),公式如下

$$
Normalized(e_i) = e_i - \frac 1 n \sum_{j = 1}^{n}e_j
$$

然后，使用创建的网络创建 `Extractor`,如果编译时 `NCNN_STRING` 宏存在，那么可以使用明文字符串读取相关信息。这里使用 `data` 获取输入信息，使用 `prob` 获取网络计算出的，关于每个类的概率。

最后是将结果输出到 `cls_scroes` 中。

这份代码使用的网络结构是之前就已经训练好的，ncnn 支持使用 caffe 和 tensorflow 已经训练好的网络结构并使用 `./tools/` 下的工具转换为 ncnn 可以使用的模型且直接加载到其网络中。