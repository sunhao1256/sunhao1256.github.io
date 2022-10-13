---
title: "Go包管理"
date: 2022-01-11T14:43:18+08:00
tags: ['学习']
---

# Go 包管理

```go
  go mod init:初始化modules
  go mod download:下载modules到本地cache
  go mod edit:编辑go.mod文件，选项有-json、-require和-exclude，可以使用帮助go help mod edit
  go mod graph:以文本模式打印模块需求图
  go mod tidy:检查，删除错误或者不使用的modules，下载没download的package
  go mod vendor:生成vendor目录
  go mod verify:验证依赖是否正确
  go mod why：查找依赖
  ​
  go test    执行一下，自动导包
  ​
  go list -m  主模块的打印路径
  go list -m -f={{.Dir}}  print主模块的根目录
  go list -m all  查看当前的依赖和版本信息
```
