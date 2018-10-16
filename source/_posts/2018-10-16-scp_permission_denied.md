---
title: scp permission denied 的原因及解决办法
category: Linux
date: 2018-10-16 17:00
---

scp 的一般格式

```shell
scp [-P {port}]  {from_user}@{from_host}:{/source/file} {to_user}@{to_host}:{/to/file}
```

scp permission denied 的原因通常是因为权限问题, 认证失败会要求输入密码

确保 `to_user` 在 `to_host` 上有以下权限

- 在 `to/file`, 对该目录有 `wx` 权限
- 如果 `to/file` 有同名文件，确保对该文件有写权限

通过以上两个措施一般可以解决 `scp permission denied` 的问题