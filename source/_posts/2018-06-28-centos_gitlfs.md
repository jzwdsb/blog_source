---
title: Centos install gitlfs
date: 2018-06-28
category: git
tags: [linux, CentOS, git]
---

git lfs (Large File Storage) 子系统将大文件(视频，可执行文件，数据集和图像等大文件)存储在远程服务器中，本地 clone 时替换为文本指针存储在 git 中，以节省网络带宽和本地硬盘资源。
所以下载一些 repository 时会遇到一些大文件没有 clone 时的情况，这时要安装 git-lfs 子系统.

```shell
sudo yum install epel-release
sudo yun install git
curl -s https://packagecloud.io/install/repositories/github/git-lfs/scrips.rpm.sh | sudo bash
sudo yum install git-lfs
git install lfs
```