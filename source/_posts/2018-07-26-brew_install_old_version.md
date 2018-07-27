---
title: Mac 上使用 brew 安装旧版本 formula
date: 2018-07-26
category: Mac
tags: [Homebrew]
---

MacOS 下 brew 更新后总是安装各种包总是最新版的, `brew install python` 总是安装 `python3.7`, 大部分包还没有做适配，安装 `tensorflow` 报错，安装 `pytorch` 报错，无奈只能安装回 `python.3.6`。

- 找到自己想要的 `formula` 版本对应的 rb 文件
  以 `python` 为例, 在
  `https://github.com/Homebrew/homebrew-core/commits/master/Formula/python.rb`
  中 commit 历史中找到自己想要的历史版本，可以根据 `comment` 找其对应的版本信息
- 选择对应的 commit, 点击 `view` 后以 `raw` 模式打开
- 将打开的链接作为新的安装包安装，需要卸载新版本，或者将其全部内容替换 `/usr/local/Homebrew/Library/Taps/homebrew/homebrew-core/Formula/` 下的同名文件，然后再安装这个 `formula`

```shell
brew uninstall python
# 直接使用链接
brew instsall https://raw.githubusercontent.com/Homebrew/homebrew-core/e128fa1bce3377de32cbf11bd8e46f7334dfd7a6/Formula/python.rb
# 覆盖 python.rb文件，再重新安装，不要更新 formula
brew install python
```

之后所有依赖 `python3.7` 的 `formula` 都会崩溃，比如 `macvim`, 依照相同方法找到旧版本 `macvim.rb` 文件重新安装，直接安装的话会更新 `python`.

没错，`brew` 放弃新版本安装旧版本就是这么麻烦。