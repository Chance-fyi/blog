---
title: "构建自己的go-gin-api脚手架(一)"
date: 2022-01-10T15:37:44+08:00
draft: false
categories: ["Go"]
tags: ["Go", "Gin"]
---

## 目标

- 学习并掌握 Go 语言
- 学习并熟练使用 Gin 框架
- 学习并熟练使用 Go 热门常用的组件
- 搭建一个自己开箱即用的 api 框架
- 锻炼自己的封装抽象能力

## 初始化

使用 Goland 创建一个新项目`go-gin-api`。

![image-20220110160124841](https://image.chance.fyi/image-20220110160124841.png)

可以看到项目中有一个 **go.mod** 的文件。

```
module go-gin-api

go 1.17
```

## Gin

### 安装

下载并安装 Gin ：

```shell
$ go get -u github.com/gin-gonic/gin
go get: added github.com/gin-contrib/sse v0.1.0
go get: added github.com/gin-gonic/gin v1.7.7
go get: added github.com/go-playground/locales v0.13.0
...
```

可以看到项目中又多了一个 **go.sum** 的文件，**go.sum** 文件详细罗列了当前项目直接或间接依赖的所有模块版本，并写明了那些模块版本的 SHA-256 哈希值以备 Go 在今后的操作中保证项目所依赖的那些模块版本不会被篡改。

**go.mod** 文件也发生了改变：

```
module go-gin-api

go 1.17

require (
	github.com/gin-contrib/sse v0.1.0 // indirect
	github.com/gin-gonic/gin v1.7.7 // indirect
	github.com/go-playground/locales v0.13.0 // indirect
	...
)
```

`indirect` 的意思是传递依赖，也就是非直接依赖。

### 测试

创建一个 **main.go** 文件

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	_ = r.Run() // 监听并在 0.0.0.0:8080 上启动服务
}
```

执行`go run main.go`命令来运行代码：

```shell
# 运行 main.go 并且在浏览器中访问 http://127.0.0.1:8080/ping
$ go run main.go
```

![image-20220110164032291](https://image.chance.fyi/image-20220110164032291.png)

我们已经使用了`github.com/gin-gonic/gin`包，可以执行`go mod tidy`命令重新整理依赖。

再次查看 **go.mod** 文件

```
module go-gin-api

go 1.17

require github.com/gin-gonic/gin v1.7.7

require (
	github.com/gin-contrib/sse v0.1.0 // indirect
	github.com/go-playground/locales v0.13.0 // indirect
	...
)

```

`github.com/gin-gonic/gin`已经变成了直接依赖。

## 参考

煎鱼的[「连载一」Go 介绍与环境安装](https://eddycjy.com/posts/go/gin/2018-02-10-install/)
