---
title: hexo next 主题添加站内搜索
category: 杂记
tags: [hexo, next]
date: 2018-07-22 17:00
---

## Next 主题添加站内搜索功能

- 安装 `hexo-generator-searchdb`, 站点根目录执行以下目录

```shell
npm install hexo-generator-searchdb  --save
```

- 编辑 `站点配置文件`, 新增以下内容到任意位置

```yml
search:
    path: search.xml
    field: post
    format: html
    limit: 10000
```

- 编辑 `主题配置文件`, 启用本地搜索功能

```yml
# Local search
local_search:
    enable: true
```

## 可能出现的问题

如果出现点击搜索一直停留在加载状态，可能是文件编码问题，可以在本地测试时访问根目录下 `search.xml` 文件

```url
localhost:4000/search.xml
```

查看报错信息