---
title: Caffe 中的 BatchNorm 实现
category: CNN
date: 2018-08-08
mathjax: true
tags: [bug]
---

这两天要修复内部框架中 BatchNorm 层在 `use_global_stat` 为 `false` 情况下与 Caffe 输出相差较大的 bug, 根据公式算法实现后仍与 Caffe 有较大差异，研究下 Caffe 中 BatchNorm 的实现，与论文中的四行公式还是有较大出入。

# Caffe 中 BatchNorm 的描述

Caffe 关于 BatchNorm 的描述可见此[链接](http://caffe.berkeleyvision.org/tutorial/layers/batchnorm.html)
以下是描述 Caffe 中 BatchNorm 层参数的 proto 文件。

```protobuf
message BatchNormParameter{
    optional bool use_global_stats = 1；
    optional float moving_average_fraction = 2 [default= = .999];
    optional float eps = 3 [default = 1e-5];
}
```

## `use_global_stat`

`use_global_stat` 为 `true` 时，表示使用全局统计量对当前 `mini-batch` 进行规范化，当为 `false` 时不使用全局统计量，而是使用当前 `mini-batch` 的均值和方差。

这次的问题主要是在模型中 `use_global_stat` 参数为 `false` 情况下导致的，内部框架没有处理 `use_global_stat` 为 `false` 的情况，默认使用全局统计量规范化输入数据，有团队发现在一些模型中该参数为 `false` 时效果更好，于是要添加这个功能。

## `moving_average_fraction`

每次迭代中滑动平均的 $\beta$ 系数。
当 $\beta$ 越小时会使全局统计量下降更快，会给最近一次算出来的均值最大的权重
每次迭代中都是用当前 `batch` 的均值 $Y_t$ 与之前的滑动均值 $S_{t-1}$ 更新当前的滑动均值。
公式如下

$$
S_t = \beta \cdot Y_t + (1 - \beta) \cdot S_{t-1}
$$

$\beta$ 即为模型参数中的 `moving_average_fraction` 参数。

原先根据[论文](https://arxiv.org/abs/1502.03167)的公式实现内部框架中 `use_global_stat` 为 `false` 的情况，发现仍与 `Caffe` 的差值较大。均值的计算方式就是简单的算术平均，看来要改下算法。

## `eps`

为防止除数为 0 加上的扰动量 $eps$, 默认为 $1\times 10^{-5}$.

# 实现

Caffe 中 BatchNorm 的实现可以定位到 `layer/batch_norm_layer.cpp` 中的 `Forward_cpu` 方法。
重点关注 mean 和 variance 的计算部分。

## 均值的计算

```C++
// compute mean
caffe_cpu_gemv<Dtype>(CblasNoTrans, channels_ * num, spatial_dim,
                      1. / (num * spatial_dim), bottom_data,
                      spatial_sum_multiplier_.cpu_data(), 0.,
                      num_by_chans_.mutable_cpu_data());
caffe_cpu_gemv<Dtype>(CblasTrans, num, channels_, 1.,
                      num_by_chans_.cpu_data(), batch_sum_multiplier_.cpu_data(), 0.,
                      mean_.mutable_cpu_data());
// subtract mean
caffe_cpu_gemm<Dtype>(CblasNoTrans, CblasNoTrans, num, channels_, 1, 1,
                      batch_sum_multiplier_.cpu_data(), mean_.cpu_data(), 0.,
                      num_by_chans_.mutable_cpu_data());
caffe_cpu_gemm<Dtype>(CblasNoTrans, CblasNoTrans, channels_ * num,
                      spatial_dim, 1, -1, num_by_chans_.cpu_data(),
                      spatial_sum_multiplier_.cpu_data(), 1., top_data);
```

以上三个 cblas 库的 gemv 分别完成的是

- $\mu_{n, c} = \frac{1}{n\cdot h\cdot w}  A_{n\cdot c, h\cdot w} \times x_{h\cdot{w},1}$
  其中由于输入为四维 NCHW 格式，所以这里计算每个 sample 的均值。输出 $out_{n,c}$ 即为第 n 个 sample 的第 c 个 channel 的均值。
  $x$ 为全一列向量，这样即完成单个 feature map 的求和。
- $s_c = \mu_{n, c}^T\times x_{n, 1}$
  求各 channel 的均值的和，这里第一个参数为 `CblasTrans` 即指定第一个矩阵要转置, $s_c$即为第 c 个 channel 的在不同 sample 上的均值
- $S = A_{n, c}\times s_c $

## 方差的计算

```C++
caffe_sqr<Dtype>(top[0]->count(), top_data,
                 temp_.mutable_cpu_data());
caffe_cpu_gemv<Dtype>(CblasNoTrans, channels_ * num, spatial_dim,
                      1. / (num * spatial_dim), temp_.cpu_data(),
                      spatial_sum_multiplier_.cpu_data(), 0.,
                      num_by_chans_.mutable_cpu_data());
caffe_cpu_gemv<Dtype>(CblasTrans, num, channels_, 1.,
                      num_by_chans_.cpu_data(), batch_sum_multiplier_.cpu_data(),
                      0., variance_mutable_cpu_data());
```

以上三个 cblas 函数调用分别完成的是

- $\sigma^2 = (X-E(X))^2$
  调用 `sqr` 计算 element wise 的 square.
- $v_c^2 = \sum_{i=1}^{m}\sigma_i$
  计算 channel 的 feature map 的 variance 的和
- $V_n^2 = \sum_{i=1}^{m}v_i$
  计算 各 channel 的 simple 的 variance 的和

## 均值滤波的计算

```C++
this->blobs_[2]->mutable_cpu_data()[0] *= moving_average_fraction_;
this->blobs_[2]->mutable_cpu_data()[0] += 1;
caffe_cpu_axpby(mean_.count(), Dtype(1), mean_.cpu_data(),
                moving_average_fraction_, this->blobs_[0]->mutable_cpu_data());
int m = bottom[0]->count() / channels_;
Dtype bias_correction_factor = m > 1 ? Dtype(m) / (m - 1) : 1;
caffe_cpu_axpby(variance_.count(), bias_correction_factor,
                variance_.cpu_data(), moving_average_fraction,
                this->blobs_[1]->mutable);
```

- $S_t=Y_t + \alpha \times S_{t-1}$
- $V_t = V_{t-1} + \alpha \times V_{t-1}$

$\alpha$ 即 `moving_average_fraction_`, 该参数从模型中读取。
多次迭代后 $S_t$ 应趋于稳定。

## 方差的归一化处理

```C++
caffe_add_scalar(variance_.count(), eps_, variance_.mutable_cpu_data());
caffe_sqrt(variance_.count(), variance_.cpu_data(),
           variance_.mutable_cpu_data());
```

$V = \sqrt{V^2 + eps}$

# 分析

Caffe 中大量调用 blas 作为底层计算，blas 的函数接口一般都比较复杂，但是掌握其命名规则和函数的公式还是很好理解的。
Caffe 中的 BatchNorm 层只对数据做了归一化处理，计算均值的方法使用了的均值滤波算法，线性变换操作放在随后的 Scale 层，可以从其模型在 netscope 的可视化模型中看出，每一个 BatchNorm 后都跟有 Scale 层。
但是内部框架的 Batchnorm 层则是在 `load_param` 方法中载入模型时，使用模型中的全局统计量计算出均值和方差后，在 `forward` 方法中实现对输入数据的线性变换。
如果修复成功 Caffe 的 BatchNorm 的输出应与内部框架的 BatchNorm 的输出相同，但是在他们的这一层的操作不一样的情况下应该如何做到这一点呢。
Caffe 是在每个 channel 的所有 sample 上做均值滤波，但是内部框架是用来做 inference 的，通常只有一个 sample 作为输入，NCHW 中的 N 一般恒为 1, 源码中其实也没有处理输入数据为 3 维以上的实现，其实 `use_global_stat` 参数根据 Caffe 中的描述也只有在为训练时才会为 `false`。
(╯‵□′)╯︵┻━┻

- [ ] 继续分析
- [ ] (╯‵□′)╯︵┻━┻ leader 说这做不到实时，毙了

~~模型又双叒叕改了~~