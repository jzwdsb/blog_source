---
title: git rebase
category: git
date: 2018-10-23
description: git 变基操作
---

通常的 workflow 是个人从 master 上拉一个分支然后自己开发相关的模块，经过数次 commit 再与 master merge.
在与 master merge 时可能出现有冲突的情况，通常是 master 分支上也有人 commit, 并且与本分支冲突，此时想要合并可以使用两种方法

# merge

```shell
git merge branch-a
```

将 branch-a 合并到当前分支中，出现的冲突需要手动解决然后再次提交

这样会使 master 分支分叉，有时希望项目的 master 保持单一的历史发展轨迹，不希望分叉，这时可以采用 rebase 操作.

# rebase

```shell
git rebase master
```

将当前分支 rebase 到 master 上，相当于将 branch-a 的最后一次 commit 的源提交修改为 master 的最后一次提交，中间也可能会有 conflict, 将 conflict 解决后可以继续 rebase 操作。

```shell
git rebase --continue
```

rebase 操作可以是整个项目的开发轨迹在一条直线上，相对于一个 merge, rebase 包括了所有的组合变化。
