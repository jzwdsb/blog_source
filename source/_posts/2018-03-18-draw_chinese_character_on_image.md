---
layout: post
category: Python
date: 2018-03-18
title: 在图像上绘制中文字体的方法
description: 使用 Pillow
---

　　在使用 opencv 的 `putText` 方法在图像上绘制汉字时发现只会打出问号，bing 了下发现 opencv 只支持 Ascii 的部分字符，不能绘制 utf8 字符，变通下，使用 python3 的图像处理库 `Pillow` 绘制汉字.

　　首先查找系统中包含的中文字体文件，ubuntu 下查看中文字体的命令为

```shell
fc-list :lang=zh-cn
```

　　从中选择一个字体，复制字体文件 ttc 的完整路径(也可以将其或自己的 ttf, ttc 字体文件直接拷贝到工作目录)，等下会用到.

　　以下就是使用 PIL 的 Image, ImageDraw, ImageFont 在 opencv 得到的图像中添加汉字了.
　　首先还是将 opencv 的图像格式转换为常用的 RGB 格式，使用 `Image.fromarray` 载入图像.<br>
　　初始化绘制字体时使用的字体对象，将刚才字体文件路径作为 `ImageFont.truetype` 的参数，第二个参数是字体的大小．<br>
　　初始化绘制时使用的 `ImageDraw.Draw` 对象，参数为绘制对象，使用之前载入的 `Image` 对象.<br>
　　使用 `ImageDraw` 对象的 `text` 方法绘制文字，参数为绘制的文字的左上角坐标，绘制字符串，字体，填充颜色.
　

```python3
import cv2
from PIL import Image, ImageFront, ImageDraw

cap = cv2.VideoCapture(0)
ret, frame = cap.read()
pil_image = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
img = Image.fromarray(frame)
font = ImageFont.truetype("/usr/share/fonts/wps-office/simhei.ttf", 24)
draw = ImageDraw.Draw(img)
draw.text((left, bottom), text, font=font, fill=(0, 0, 255))
frame_with_text = numpy.array(img)
```