---
layout: post
category: Python
date: 2018-03-14
title: 在 PyQt5 中使用 opencv 播放视频流
description: 问题在于图像格式
tags: [Python, Opencv]
---

　　现在做的应用需要结合 opencv 和 PyQt5, 但是两个库之间并没有一个明确的互相读取和调用的接口.
　　PyQt5 读取 opencv 的原始图像数据需要经过一些转换步骤.

　　由于历史原因，opencv 使用的像素格式为 BGR 格式，现在的图像和视频大都是 RGB 格式，所以使用 QImage 加载 opencv 原始数据前，需要使用 `cv2.cvtColor` 转换数据格式到 RGB 格式，参数为 `cv2.COLOR_BGR2RGB`.
　　使用 `QImage` 从 opencv `mat` 中读取图像原始数据，这里需要指定图像矩阵的数据格式和图像的宽度和高度，上面将像素格式转换为 `RGB`，图像深度为 8 位，所以格式为 `RGB888`.
　　这里播放视频流是使用的 QLabel, 设置其 Pixmap 以显示图像，定义一个 QTimer, 时间间隔为 33ms, 将其 `timeout` 信号连接到自定义函数 `play_video` 上，实现视频流的播放.

　　大致实现代码

```python
import cv2
from PyQt5 import QtCore, QtGui, QtWidgets
from PyQt5.QtCore import QTimer, pyqtSlot, pyqtSignal
from PyQt5.QtGui import QPixmap, QImage
from PyQt5.QtWidgets import QMainWindow, QLabel, QPushButton

class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.cap = cv2.VideoCapture(0)
        self.timer = QTimer()
        self.video = QLabel()
        self.timer.start(33)
        self.timer.timeout.connect(self.play_video)
    
    @pyqtslot(name="play_video")
    def play_video():
        ret, frame = self.cap.read()
        if ret:
            rgbframe = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            connvert_to_qt = QImage(rgbframe, rgbframe.shape[1], rgbframe[0], 
                                    QImage.Format_RGB888)
            self.video.setPixmap(QPixmap.fromImage(connvet_to_qt))
```

<!--
能说什么呢
-->