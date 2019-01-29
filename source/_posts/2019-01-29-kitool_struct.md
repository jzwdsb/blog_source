---
title: kitool 工具及其生成
category: 后端
date: 2019-01-29
---

kitool 工具生成的如下文件和目录

- `kite.go`
  生成工具生成，不可以改动
- `server.go`
  可以根据需要改动
- `handler.go`
  应该由开发者实现内部对应的方法逻辑
- `build.sh`
  编译打包的辅助文件，一般不需要修改
- `thrfit_gen`
  thrift 生成的 go 代码
- `clients`
  生成的客户端代码
- `conf`
  项目的基础配置文件，根据需要修改
- `script`
  上线运行必须辅助文件，一般不需要修改