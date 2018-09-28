---
title: ubuntu 下sysctl.conf 的恢复方法及 etckeeper 的简单使用
category: Linux
date: 2018-09-28 17:00
---

一时手贱覆盖了 `/etc/sysctl.conf` 文件的全部内容，虽然恢复后发现全是注释，还是后怕。
今后要注意对 `/etc` 下所有配置文件的备份。

## sysctl.conf 的恢复

/etc/sysctl.conf 属于 `procps` 软件包，删除原 `sysctl.conf` 重新安装 `procps` 恢复默认 `sysctl.conf` 配置。
可以使用 `sysctl -a` 查看当前所有 `sysctl` 配置信息，写入 `/etc/sysctl.conf` 中，原理上效果一样，存在部分语法问题。

```shell
# remove the bad sysctl.conf so it will restore the origin sysctl.conf file
sudo mv /etc/sysctl.conf /etc/sysctl.conf.bad
sudo apt -o Dpkg::Options::="--force-confmiss" install --reinstall procps
```

## etckeeper

使用 `etckeeper` 备份 `/etc` 文件

### 安装

```shell
sudo apt install git-core etckeeper
```

### 使用

```shell
sudo etckeeper init
```

查看 `/etc/etckeeper/etckeeper.conf`, 将前五行设置为如下

```plain
# The VCS to use.
# VCS="hg"
VCS="git"
# VCS="bzr"
# VCS="darcs"
```

执行如下命令

```shell
sudo etckeeper init
```

理应输出

```plain
Initialized empty Git repository in /etc/.git/
```

如果没有输出，执行以下命令再次重试

```shell
sudo etckeeper uninit
```

然后第一次提交

```shell
sudo etckeeper commit "Initial commit"
```

今后每次修改 `/etc` 下面的配置文件就需要提交一次以作记录，方便出现问题时回滚, VCS 使用的 git, 回滚方式与 git 相同。 