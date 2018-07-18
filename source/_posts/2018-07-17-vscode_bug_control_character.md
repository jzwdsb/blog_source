---
title: vscode 输入 non-printable 字符的 bug
date: 2018-07-17 18:00
category: 杂记
tags: [vscode, bug]
---

目前使用 hexo 时常提示某行存在 `non-printable character`, 不可打印字符肉眼也看不到，只能将其提示的整行删除重新输入，google 下发现这是 vscode 的bug, 已经有人提出了 [issue](https://github.com/Microsoft/vscode/issues/37114)。

这个 bug 尚未修复，目前比较可行的解决办法是在用户首选项中添加如下一行

```json
"editor.renderControlCharacters": true
```

这行配置指示编辑器渲染控制字符，这样就可以看到 `non-printable character`, 将其删除即可.