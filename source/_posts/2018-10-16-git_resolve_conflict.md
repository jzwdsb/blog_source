---
title: git pull 时遇到 conflict 以及如何 reslove
category: git
date: 2018-10-10
description: 以及如何将 fork 之后的 repo 与原 repo 保持同步
---

## 与原 repo 保持同步

在 fork 之后的 repo 目录执行以下命令，将原 repo 的 remote 地址添加到当前 git 的 upstream 中.
之后，执行 fetch 拉取，merge 合并 master.
或者执行 pull, 等价于以上两个命令同时执行
```
# It doesn't matter how to name the remote, upstream for example
git remote add upstream git://github.com/origin_repo/repo.git

git fetch upstream
git merge upstream/master master

# or a simple way
git pull upstream master
```

## resolve conflict

在 pull 操作后可能会产生冲突，通常是因为与某个合作者的共同修改了同一文件的同一行导致，又或者因为本地已经落后于远端。

使用 `git status` 查看当前冲突的文件，在文件冲突的地方会有如下的格式

```plain
If you have questions, please
<<<<<<< HEAD
open an issue
=======
ask your question in IRC.
>>>>>>> branch-a
```

保存修改

```shell
git add .
```

查看当前的冲突并决定要保存哪一个，之后 commit 提交后解决冲突。

```shell
git commit -m "conflict resolved"
```

最后 push 到自己 fork 之后的 repo

```shell
git push origin master
```