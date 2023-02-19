---
title: "构建自己的go-gin-api脚手架(四)"
date: 2022-10-16T14:14:26+08:00
draft: true
categories: ["Go", "go-gin-api"]
tags: ["Go", "Command", "Cobra"]
---

在我设计这个脚手架中，最终的功能肯定不止一个 web 服务，还会包括一些数据库迁移、代码生成等功能。所以需要多个命令来区分运行不同的功能。

## Cobra

> 地址：[https://github.com/spf13/cobra](https://github.com/spf13/cobra)![](https://img.shields.io/github/stars/spf13/cobra?style=social)

Cobra 与 Viper 出自同一个作者之手，是一个用于创建 CLI（命令行界面）的应用程序。有许多大型项目使用了它，例如 Hugo、Docker、Kubernetes、Etcd 等等。

### 概念

Cobra 基于三个基本概念 commands，arguments 和 flags。其中 commands 是命令，arguments 是参数，flags 则是对命令的一种扩展。

### 安装

```shell
go get -u github.com/spf13/cobra@latest
```
