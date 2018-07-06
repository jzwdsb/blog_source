---
title: NCNN 源码剖析(五) Extractor
description: NCNN 中的 Extractor
date: 2018-06-27 19:00
category: CNN
tags: [CNN, ncnn]
---

`Extractor` 类声明在 `net.h` 中

## 接口声明

### 公开接口

`Extractor` 公开接口如下

```C++
void set_light_mode(bool enable);
void set_num_threads(int nums_threads);
#if NCNN_STRING
int input(const char* blob_name, const Mat& in);
int extract(const char* blob_name, Mat& feat);
#endif
int input(int blob_index, const Mat& in);
int extract(int blob_index, Mat& feat);
```

公有接口的主要包含以下内容

- `set_light_mode`
    设置轻量模式，在轻量模式下中间层的 `blob` 会被回收
- `set_num_threads`
    设置并行时的线程数，默认线程数依赖于系统
- `input`
    根据指定 `blob` 喂给所绑定的 `Net`
- `extract`
    根据指定的 `blob` 获取相应的结果

### 成员属性

`Extractor` 中的成员属性如下

```C++
const Net* net;
std::vector<Mat> blob_mat;
bool lightmode;
int numthread;
```

- `net`
    指明创建这个 `Extractor` 的 `Net` 对象
- `blob_mat`
    `blob_mat` 通常表示层与层之间传递的数据

## 源码

### 接口声明源码

```C++
class Extractor
{
public:
    // enable light mode
    // intermediate blob will be recycled when enabled
    // enabled by default
    void set_light_mode(bool enable);

    // set thread count for this extractor
    // this will overwrite the global setting
    // default count is system depended
    void set_num_threads(int num_threads);

#if NCNN_STRING
    // set input by blob name
    // return 0 if success
    int input(const char* blob_name, const Mat& in);

    // get result by blob name
    // return 0 if success
    int extract(const char* blob_name, Mat& feat);
#endif // NCNN_STRING

    // set input by blob index
    // return 0 if success
    int input(int blob_index, const Mat& in);

    // get result by blob index
    // return 0 if success
    int extract(int blob_index, Mat& feat);

protected:
    friend Extractor Net::create_extractor() const;
    Extractor(const Net* net, int blob_count);

private:
    const Net* net;
    std::vector<Mat> blob_mats;
    bool lightmode;
    int num_threads;
};
```