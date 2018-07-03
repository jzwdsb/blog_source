---
layout: post
category: git
date: 2018-02-24
title: git clone 时遇到 early eof 时的解决办法
---

　　git clone 自己一个 repository 时速度不仅奇慢而且到中间常出现如下错误．

```shell
error: RPC failed; curl 56 GnuTLS recv error (-54): Error in the pull function.
fatal: The remote end hung up unexpectedly
fatal: early eof
fatal: index-pack failed
```

　　经过[搜索](https://stackoverflow.com/questions/17683295/git-bash-error-rpc-failed-result-18-htp-code-200b-1kib-s)发现是 curl 下载时的 buffer 过小导致的，这个 repository 确实有比较大的文件．所以更改下 postbuffer 的大小．

```shell
git config --global http.postbuffer 24288000
git config --global https.postbuffer 24288000
```

　　以上是针对 git 的第一条错误提示搜索的解决办法，同样的仅针对 early eof 错误而言，也有解决办法．<br>
　　从 remote host 上下载 repository 时服务器端会默认先压缩目标文件再进行传输，客户端再解压．压缩指数范围为 [-1, 9], -1 以 zlib 为默认压缩库, 0 不压缩, 1 ... 9 为压缩速度与最终文件大小的权衡，数字越大，压缩越慢，但最终获得的目标文件也就越小.<br>
　　[stackoverflow](https://stackoverflow.com/questions/2505644/git-checking-out-problem-fatal-early-eofs)上有人测试发现关掉压缩功能或者设置为 -1 能够避免出现 early eof.

```shell
# 关闭压缩功能
git config --global --all core.compression 0
# 设置为 -1 
git config --global --all core.compression -1
```