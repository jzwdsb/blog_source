---
title: 关于文件的读写问题
date: 2019-03-18
category: Operate System
---

# 问题

现在遇到一个问题，在在一个函数中向文件写内容并返回该文件句柄, 在上游函数中该文件句柄中读取，读不到内容，原来猜测是没有 flush 到 disk 导致的。

可是调用 `file.Sync` 显式 flush 到 disk 仍然没有作用。

# 原因

意识到文件在写操作完成后 file cursor 已经定位到文件末尾，所以直接读取只会得到 eof.

# 解决办法

将 file cursor 重新定位到文件开头

```golang
offset, err := file.Seek(0, 0)
if offset != nil || err != nil {
    // seek to file begin failed.
    // handle error
}
```