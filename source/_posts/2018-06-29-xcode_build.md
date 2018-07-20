---
title: 在 Mac 下使用 CMake 编译 IOS 项目时遇到的问题
date: 2018-06-29
category: Mac
---

CMAKE 编译 IOS 项目时报错提示 `CMAKE_OSX_SYSROOT` 路径不存在.
`CMAKE_OSX_SYSROOT` 用来指定开发系统的 SDK，如果该环境变量不存在的话就会提示这样的错误。在 MAC 上安装完 Xcode 后通常会自动设置。

可以通过以下命令查看开发使用的 SDK 相关信息

```shell
xcodebuild -version -sdk iphoneos
```

如果报错提示 `error select active develop diectory`

则需要手动设置 Xcode 的路径

```shell
sudo xcode-select -s Applications/Xcode.app/Contents/Developer/
```