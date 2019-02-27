---
title: next 主题背景添加 canvax next 特效
category: blog
date: 2019-02-27 20:00
tags: [hexo, next]
---

之前记得看到有些博客背景有可以用鼠标吸引的几何线条，很是喜欢，一直想加到自己博客里面。

theme 是 nexT, 在 `themes/next/layout/_layout.swig` 的 body 块中添加如下内容即可

```swig
<script type="text/javascript" color="0,0,0" opacity='0.5' zIndex="-1" count="99" src="//cdn.bootcss.com/canvas-nest.js/1.0.0/canvas-nest.min.js"></script>
```