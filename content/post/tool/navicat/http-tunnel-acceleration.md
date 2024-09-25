---
title: "Navicat HTTP tunnel 加速"
date: 2024-09-14T13:13:19+08:00
draft: false
categories: ["Navicat"]
tags: ["Navicat", "http-tunnel"]
---

## 背景

在公司环境中，测试和生产数据库通常不允许直接外网访问。为了使用 Navicat 管理这些数据库，我们需要通过隧道连接。我们选择使用 HTTP 隧道，Navicat 安装目录下提供了相应的 PHP 隧道脚本，只需将其部署到可以连接数据库的服务器上即可。

然而，在实际使用过程中，我发现 HTTP 隧道会显著降低操作速度，严重影响工作效率。

## 问题分析

通过查看 PHP 隧道脚本的代码，我了解到其工作原理：Navicat 将 SQL 语句发送给 PHP 脚本，PHP 执行 SQL 并返回结果。这个中转过程会引入一些延迟，但实际体验中的延迟似乎过高。

为了深入分析这个问题，我决定使用 Wireshark 进行抓包。我设置了以下过滤器来捕获所有通过 HTTP 隧道的 Navicat 请求:

```
http.request.full_uri == "http://your-domain.com/ntunnel_pgsql.php" and http.request
```

我执行了一系列常见操作:

1. 打开数据库
2. 查看一个表结构
3. 查看一个表数据
4. 按 id 倒序筛选数据

![picture 0](http://image.chance.fyi/image-2024092510075267395.png)

抓包结果显示，这些基本操作总共触发了 177 次请求。虽然每个请求的详细内容不方便都截图出来，但是通过观察每个请求的 Length，也可以发现存在大量重复请求。

## 解决方案

最初，我尝试在 Navicat 设置中寻找可以缓存这些数据的选项，以减少重复请求。但是没有找到相关设置。

因此，我决定开发一个本地运行的代理程序，用于缓存重复请求，从而减少网络请求次数，提高操作速度。

我使用 Go 语言开发了一个代理工具 [navicat-http-tunnel-acceleration](https://github.com/Chance-fyi/navicat-http-tunnel-acceleration)。它的使用非常简单:

1. 克隆仓库到本地
2. 在项目目录下运行 `go run . --url http://your-domain.com/ntunnel_pgsql.php --port <your-port>`
3. 将 Navicat 的 HTTP 隧道设置成 `http://127.0.0.1:<your-port>`

该工具接受两个参数:

- `--url`: HTTP 隧道的地址
- `--port`: 服务监听的端口

运行后，你可以通过访问 `http://127.0.0.1:<your-port>/sql` 接口查看每个 SQL 的请求次数。根据这些信息，你可以在 `config.go` 文件中添加需要缓存的 SQL。

## 使用效果

通过使用这个代理工具，成功减少了重复请求，显著提升了 Navicat 操作的响应速度，可以减少大约 2/3 的响应时间。
