---
layout: post
category: Python
date: 2018-03-09
title: face_recogition 初探
description: 尝试制作一个小玩具
tags: [Python]
---

　　这次要做一个人脸支付系统(其实主要是识别出人物的身份以便从数据库中查找数据)，尝试使用 python3 + [face_recognition](https://github.com/ageitgey/face_recognition)  

　　face_recognition 依赖于 [dlib](https://github.com/davisking/dlib.git). 这是个现代 C++ 的库, 集成了机器学习算法和用来创建解决现实问题的复杂软件的工具.

## dlib

```shell
# 依赖 boost
sudo apt install --no-install-recommends  libboost-all-dev

# clone dlib 源代码
git clone https://github.com/davisking/dlib.git

# 开始编译
cd dlib
mkdir build
cd build
cmake .. -DUSE_AVX_INSTRUCIONS=1
cmake --build .

# 编译 python api
cd ..
# 如果 cpu 支持 avx 的话后面的选项可以打开，会快点
python setup.py --yes USE_AVX_INSTRUCTION
```

## 安装 face_recognition

```shell
sudo -H pip3 install face_recognition
```

## 简单使用

### 命令行接口

　　当 `pip3 install face_recognition` 成功后，会得到两个简单的命令行程序.

1. `face_recognition` 在一张照片或者全是相片的文件夹中识别人脸.
2. `face_detection` 在一张照片中或者全是相片的文件夹中探测人脸.

　　首先，要识别人脸必须提供一个文件夹，里面包含已知的人的照片，应该是一个人物一张相片，文件名即为该人物的标签．<br>
　　然后，选择另外一个文件夹，里面是要识别的照片.

　　如果识别结果中有识别错误的情况，比如两个长相相似的人识别成了同一个人，可以调整 `--tolerance` 参数，默认的 `tolerance` 值为 0.6, 可以调低这个值使脸部比较更严格．

```shell
face_recognition --tolerance 0.54 ./pictures_of_people_i_know/ ./unknown_pictures/
```

### python API

　　详细的 python API 可见[官方文档](https://face-recognition.readthedocs.io/en/latest/face_recognition.html).

　　简单使用

```python
#! /usr/bin/python3
#  -*- coding:utf8 -*-

import cv2
import face_recognition

cap = cv2.VideoCapture(0)

cv2.namedWindow("video", 0)

'分辨率太高会导致视频卡顿，可以降低分辨率至 320 * 240, 以得到流畅效果'
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 320)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 240)
while True:
    ret, frame = cap.read()
    face_locations = face_recognition.face_locations(frame)
    for face in face_locations:
        top, right, bottom, left = face
        cv2.rectangle(frame, (left, top), (right, bottom), (255, 0, 0), 3)
    cv2.imshow("video", frame)
    if cv2.waitKey(20) == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
```

　　这段代码的作用是打开摄像头后在获取的每个图像中识别出人脸并画上红色框

<!--
> 这就是生活，生活总是这样，如湍流般激流奔涌，年轻的时候往往不知道做什么好，被生活裹着就走了.
-->