---
layout: post
title: "如何开发 fork 的 Golang 项目"
subtitle: ''
author: "yqsas"
header-style: text
catalog: false
tags:
  - Golang
---

## 如何开发 fork 的 Golang 项目

### 问题

Go 基于位置的包导入机制，使得我们自己 fork 下来的项目进行二次开发时，会发现 fork 中导入的包路径依旧是源项目路径。于是我如果要正常运行的话就得把对应路径改成自己的 fork repo，接着开发完做 pull request 的时候又得修改回来，这显然是一个错误的做法。

### 解决方式

1. 打开心仪的源 repo 地址，点击 fork 按钮

2. go get 源 repo

   ```bash
   go get github.com/xx/xxxx
   ```

3. 进入目录

   ```bash
   cd $GOPATH/src/github.com/xx/xxxx
   ```

4. 添加 fork 仓库源并更新

   ```bash
   git remote add fork git@github.com:xx/xxxx.git

   git fetch fork
   ```

5. 设置当前分支跟踪远程的 fork 分支

   ```bash
   git branch -u fork/master
   ```

6. 在 ide 打开这个目录开始愉快地开发吧 :)

### 参考

> [Working with forks in Go](https://dev.to/loderunner/working-with-forks-in-go-3ab6)