---
title: hexo generate 报 `Cannot set property 'lastIndex' of undefined` 错误及其解决方法
date: 2018-06-30
tags: [hexo, bug]
---

  今天使用 hexo 生成 html 文件时报 `Cannot set property 'lastIndex' of undefined` error。 这个问题在 github 上已经有人提出了 [Issue](https://github.com/hexojs/hexo/issues/3128)

  解决方式是将 `_config.yml` 文件中的 `auto_dectet` 属性设置为 `false`.

  现在发现是自己写 Markdown 时内嵌代码后面的 \`\`\` 后面多打一个字符导致定界错误，将其删除且将 `auto_dectet` 设置回 `true` 后显示正常。