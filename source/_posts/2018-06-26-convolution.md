---
title: 卷积神经网络的原理及实现
description: 今天的笔记，记得很是凌乱
category: 神经网络
date: 2018-06-26
mathjax: true
tags: [CNN]
---

## 卷积操作

  卷积的主要目的是从输入图像中提取特征，卷积操作通过过滤器学习图像的特征，保留图像中像素的空间关系。
  过滤器不同的值对卷积的结果有不同的影响。过滤器的值不同可以提取不同的特征，比如边缘检测，锐化，模糊，高斯模糊等。

![filter_values](\image\filter_values.png)

## 线性矫正单元

  在每一次卷积操作后，都会有一步称为线性矫正单元(Rectified linear Unit, ReLU)操作，ReLU 并不是一个线性操作，它的输出是根据如下公式给出

$$
Output = max(0, Input)
$$

  ![ReLU](/image/ReLU.png)

## 卷积层的前向计算

  卷积层的输入多来自与输入层和 pooling 层,在每一层中卷积核的大小相同。

  ![cnn_layers](/image/cnn_layers.png)

  卷积层2的 map 是不同卷积核在前一层每个 map 上进行卷积，并且将每个对应位置上的值相加再加上一个偏置项。

  ![convolution_op](/image/convolution_op.png)

  每次用卷积核与 map 对应位置相乘，然后滑动卷积核进行下一个神经元的计算。

  假设输入 map 为 $w \times h$ 大小，卷积核大小为 $m \times n$. 滑动步长为 s (s < m, s < n), 输出图像的尺寸应为
  
  $$
  width = \frac {(w - m + 1)} s   \\、
  height = \frac {(h - n + 1)} s  \\\
  $$

## 卷积层的后向计算

在反向传播过程中，如果第 x 层的 a 节点通过权值 W 对 x + 1 层的 b 节点有贡献，则在反向传播过程中，梯度通过权值 W 通过节点 b 传播回节点 a.

在上图中, $A_{1, 1}$ 通过权重 $B_{1, 1}$ 与 map 中的 $C_{1, 1}$ 相关联,而 $A_{1,2}$ 与 C 中两个元素相关联，分别与权重 $B_{1,2}$ 对应于 $C_{1,1}$, 与权重 $B_{1, 1}$ 对应于 $C_{1,2}$.

可以使用另外一种简单的方法实现这样简单的关联.通过将卷积核顺时针旋转 180 度(与转置操作不同，主对角线上的元素也经过旋转),与扩充后的梯度矩阵进行卷积.

扩充的方法如下: 如果卷积核为 $k \times k$,待卷积矩阵为 $n \times n$, 需要以 $n \times n$原矩阵为中心扩展为

$$
(n + 2 (k - 1)) \times (n + 2(k - 1))
$$

假设 D 为反向传播到卷积层的梯度矩阵，则 D 应与矩阵 C 大小相等, 这里为 $3 \times 3$. 我们首先需要将它扩充至 $5 \times 5$ 大小的矩阵.

同时将卷积核与扩充后的梯度矩阵进行卷积.

## 卷积层的计算复杂度

卷积的计算复杂度基于三个方面

1. 卷积操作(convolution operation)
2. 卷积核尺寸(kernel size)
3. 缓存友好(cache friendly)

首先在一个卷积核中，对于每个输出的计算量与卷积核的大小是二次方关系。所以相比较于$5 \times 5$的卷积核, $7 \times 7$ 卷积核的计算量大概是其两倍。

其次，典型的 CNNs 使用的卷积核的大小从 $3 \times 3$ 到 $9 \times 9$ 不等，卷积操作通常采用嵌套的四层循环实现，外面两层是对输入图像的 x 方向和 y 方向上的迭代，内层是对卷积核 x 方向和 y 方向的迭代。卷积核较小时内层循环就会因为频繁执行 JMP 指令而效率降低。

最后，前向传播，后向传播对于输入图像和卷积核都需要行优先和列优先的存取. 目前的 2-D 图像(包括卷积核)普遍采用以行优先的方式存储在连续的内存块中，所以列优先的访问会导致高的 cache miss.

## caffe 中前向传播的实现

```C++
void BaseConvolutionLayer<Dtype>::forward_gpu_gemm(const Dtype* input, const Dtype* weights, Dtype* output, bool skip_im2col) {
  const Dtype* col_buff = input;
  if (!is_1x1_) {
    if (!skip_im2col) {
      conv_im2col_gpu(input, col_buffer_.mutable_gpu_data());
    }
    col_buff = col_buffer_.gpu_data();
  }
  for (int g = 0; g < group_; ++g) {
    caffe_gpu_gemm<Dtype>(CblasNoTrans, CblasNoTrans, conv_out_channels_ /
        group_, conv_out_spatial_dim_, kernel_dim_ / group_,
        (Dtype)1., weights + weight_offset_ * g, col_buff + col_offset_ * g,
        (Dtype)0., output + output_offset_ * g);
  }
}
```

首先，使用 `conv_im2col_gpu` 将输入特征图扩展至矩阵 B, 具体形式可见于[此链接](https://petewarden.com/2015/04/20/why-gemm-is-at-the-heart-of-deep-learning/), 这里的每一列存储的是输入特征图中各方块与各个过滤器卷积的结果，所以列数一共是 $output\\_width \times output\\_height$.

其次，使用 `caffe_gpu_gemm` 将矩阵 B 与矩阵 A 相乘，得到输出特征图，$C = A \times B$, C 的每一行都是输出特征图(output feature map), 权重矩阵 A 中，每一行都是过滤器(filter)

## caffe 中反向传播的实现

```C++
void ConvolutionLayer<Dtype>::Backward_gpu(const vector<Blob<Dtype>*>& top,
      const vector<bool>& propagate_down, const vector<Blob<Dtype>*>& bottom) {
  const Dtype* weight = this->blobs_[0]->gpu_data();
  Dtype* weight_diff = this->blobs_[0]->mutable_gpu_diff();
  for (int i = 0; i < top.size(); ++i) {
    const Dtype* top_diff = top[i]->gpu_diff();
    // Bias gradient, if necessary.
    if (this->bias_term_ && this->param_propagate_down_[1]) {
      Dtype* bias_diff = this->blobs_[1]->mutable_gpu_diff();
      for (int n = 0; n < this->num_; ++n) {
      //对一个batch中每一个map，计算其偏置的偏导
        this->backward_gpu_bias(bias_diff, top_diff + n * this->top_dim_);
      }
    }
    if (this->param_propagate_down_[0] || propagate_down[i]) {
      const Dtype* bottom_data = bottom[i]->gpu_data();
      Dtype* bottom_diff = bottom[i]->mutable_gpu_diff();
      for (int n = 0; n < this->num_; ++n) {
        // gradient w.r.t. weight. Note that we will accumulate diffs.
        if (this->param_propagate_down_[0]) {
          this->weight_gpu_gemm(bottom_data + n * this->bottom_dim_,
              top_diff + n * this->top_dim_, weight_diff);
        }
        // gradient w.r.t. bottom data, if necessary.
        if (propagate_down[i]) {
          this->backward_gpu_gemm(top_diff + n * this->top_dim_, weight,
              bottom_diff + n * this->bottom_dim_);
        }
      }
    }
  }
}
```

## gemm 的含义

gemm (general Matrix to Matrix Multiplication) 通用矩阵乘法

公式为

$$
C = \alpha \times A \times B + \beta \times C
$$

## caffe 中的 gemm 实现

```C++
/*
C=alpha*AB+beta*C
As cublas follows fortran order(colunm first order), this function adapts it to C/C++ (row first order) 
*/
template <>
void caffe_gpu_gemm<double>(const CBLAS_TRANSPOSE TransA,
    const CBLAS_TRANSPOSE TransB, const int M, const int N, const int K,
    const double alpha, const double* A, const double* B, const double beta,
    double* C) {
  // Note that cublas follows fortran order.
  int lda = (TransA == CblasNoTrans) ? K : M;
  int ldb = (TransB == CblasNoTrans) ? N : K;
  cublasOperation_t cuTransA =
      (TransA == CblasNoTrans) ? CUBLAS_OP_N : CUBLAS_OP_T;
  cublasOperation_t cuTransB =
      (TransB == CblasNoTrans) ? CUBLAS_OP_N : CUBLAS_OP_T;
  //C=AB <==> C'=B'A'
  //As fortran order and C/C++ order are transposingly related, transposing is done intrinsically
  CUBLAS_CHECK(cublasDgemm(Caffe::cublas_handle(), cuTransB, cuTransA,
      N, M, K, &alpha, B, ldb, A, lda, &beta, C, N));
}
```
<!--
## 会议记录

总结上周工作，armv7 的代码编写已完成。

尝试不同方式实现层融合。

内存池优化方案相关。

单线程优化方案已经落地

1. 不同网络对内存的需求不同
2. 网络初始阶段内存需求为小内存，后期会逐渐增长，内存池的效果几乎不存在，存在大量内存碎片
3. 卷积优化, 主要针对 gemm 进行优化
4. 框架与算子需要解耦, 否则对后期迭代开发不利
5. 框架会在几周内开始
6. 层融合优化方案
7. 内存优化
8. 使用 ntnn 中内存复用方法
9. 使用 ntnn 层中释放内存
10. 反卷积层中原先intel caffe 中使用 openmp 实现，改为使用 C++ 实现
11. 底层代码需要做混淆，否则逆向容易得到模型
12. Intel 两款 CPU 已到，测试性能
-->

## ncnn 框架结构

ncnn 的源码在 [github](https://github.com/Tencent/ncnn) 上。

当前的目录结构如下

- ./

  顶层目录包含LICENSE, README,CMakeLists.txt, build.sh 等配置信息
- ./examples/

  该目录下包含一个使用 squeezenet 做图像分类的 C++ 例子程序
- ./src/

  - 顶层目录下是一些基础代码，如宏定义，平台检测，mat 数据结构，layer 定义，blob 定义，net 定义等
  - ./src/layer 下是所有的 layer 的定义代码
  - ./src/layer/arm 是 arm 下的加速计算的 layer
  - ./src/layer/x86 是 x86 下的加速计算的 layer
- ./tools/
  该目录下是 ncnn 转换 caffe, tensorflow 模型的工具代码

设计框架大致为

- blob 存储数据
- layer 为计算单元
- network 为调度单元

ncnn 中还有 Extractor 的概念，Extractor 可以看作 Network 对用户的接口，NetWork 一般模型只需要一个实例，但是 Extractor 可以有多个实例，这样在进行多个任务时可以节省内存(模型定义模型参数不需要多个拷贝)

## 今日总结

在继续阅读部分 ncnn 源码后发现对基础理论还是有欠缺，所以又补充了关于卷积层的前向传播和后向传播方面的知识。

1. 看了 caffe 中前向传播和后向传播的实现源码
2. 阅读部分相关论文，卷积层的计算复杂度主要受三个因素影响，卷积计算，卷积核尺寸和缓存友好性
3. 熟悉 ncnn 的框架结构

## 明日计划

继续熟悉 ncnn 框架，理解 Issues 106