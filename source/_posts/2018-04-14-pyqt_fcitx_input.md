---
layout: post
category: Python
date: 2018-04-14
title: 在 PyQt5 中使用 fcitx 输入法
---

　　这次用 PyQt5 写的玩具没法用 fcitx 输入法，个人感觉和 Qt 原因相同都是缺少平台输入上下文的动态链接库，试了下，问题解决，简单记下解决办法.

```shell
sudo cp /usr/lib/x86_64-linux-gnu/qt5/plugins/platforminputcontexts/libfcitxplatforminputcontextplugin.so \
/usr/local/lib/python3.5/dist-packages/PyQt5/Qt/plugins/platforminputcontexts
```

　　源文件为 ubuntu 系统中 fcitx 的输入环境上下文的动态链接库，将其拷贝到 PyQt5 下面的 `plugins/platforminputcontexts/` 下即可，同样的, 下载的 Qt 同样默认也不带 fctix 链接库，解决方法相同. 