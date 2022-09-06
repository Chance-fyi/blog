---
title: "多个docker-compose项目之间通信与环境架构"
date: 2022-06-25T20:44:54+08:00
draft: false
categories: ["Docker"]
tags: ["Docker", "docker-compose", "Network"]
---

在讲多个 docker-compose 项目之间通信之前，想要先说一说我从刚使用 Docker 到目前为止遇到的一些问题与想法。
在刚学习 Docker 的时候，我相信很多人都跟我一样有过这种想法：我们的代码运行环境、MySQL、Redis 等服务是放到一个容器里面呢，还是放到多个容器里面呢？经过一番学习知道，Docker 官方是推荐将这些环境放置到多个容器中的，至于为什么就不详细说了，可以自行百度学习一下。
但是当放到多个容器中，又会出现一个问题，我的 API 服务该怎么访问 MySQL、Redis 呢？有三种办法可以解决。

1. 通过容器内 ip + 映射出的端口进行访问（不推荐）
2. 通过 `--link` 链接另一个容器访问(不推荐)
   `--link` 只能在 `docker run` 一个容器时链接另一个已运行的容器，所以说该方法只能单向访问。
3. 通过 `network` 将两个容器链接到同一个 `network` 中可以进行双向访问

这么看来第三种方式是最好的解决方案，但是当我们容器越来越多时，需要记住所有容器的 run 指令，将一个个的容器 run 起来，然后链接到同一个 network 中去，操作起来非常的麻烦。所以我更推荐使用 `docker-compose` 来编排管理多个容器。下面来编写一个 `docker-compose.yml` 文件演示一下。
因为我们只是来演示一下容器之间的通信，使用 MySQL、Nginx 等容器反而更加的麻烦，所以我们使用 Golang 简单写一个接口用来构建一个演示的镜像。

```go
package main

import (
	"flag"
	"github.com/gin-gonic/gin"
)

func main() {
	name := flag.String("name", "", "")
	flag.Parse()
	r := gin.Default()
	r.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"name": name,
		})
	})
	r.Run()
}
```

很简单的一段代码，运行时传入一个 name 作为服务的名字，然后使用 Gin 监听 http 请求并将服务名返回。
接下来使用 Dockerfile 构建一个镜像：

```dockerfile
FROM golang:1.18 as builder

ENV GO111MODULE=on \
    GOPROXY=https://goproxy.cn,direct

WORKDIR /home

COPY . .

RUN GOOS=linux GOARCH=amd64 go build -tags netgo .


FROM busybox

WORKDIR /home

COPY --from=builder /home/docker .

ENTRYPOINT ["./docker"]
```

这个 Dockerfile 也很简单，使用分阶段构建，首先先将 Golang 的代码打包成一个可执行文件，然后以 busybox 为基础镜像和打包的可执行文件构建成一个新的镜像，并运行我们所写的代码。
接下来我们就可以编写 `docker-compose.yml` 进行演示了：

```yaml
version: "3.8"
services:
  nginx:
    build: .
    container_name: nginx
    command:
      - -name=nginx
  mysql:
    build: .
    container_name: mysql
    command:
      - -name=mysql
  go:
    build: .
    container_name: go
    command:
      - -name=go
```

起了三个服务，分别是 `nginx` 、`mysql` 和 `go`，然后通过 `command` 追加一个 name 的参数将服务名传递进去。接下来运行 `docker-compose up -d` 看一下效果：
![图 1](http://image.chance.fyi/image-2022082716563015489.png)
可以看到 `docker-compose` 自动创建了一个 network ，并且将三个服务都加入了进来，我发现很多人写 yaml 文件的时候都会手动创建一个 network，其实这是没有必要的，因为 `docker-compose` 会自动创建一个默认的。
先进入 go 的容器里请求其他服务看一下：
因为 busybox 镜像中没有 curl ，可以使用 wget 来代替 curl 请求接口，通过服务名来访问其他的服务。

```bash
wget -q -O - http://mysql:8080 && echo
```

![图 2](http://image.chance.fyi/image-2022082722370208534.png)
![图 3](http://image.chance.fyi/image-2022082722384267098.png)
可以看到两个服务之间是双向联通的，可以互相访问，访问不存在的 Redis 服务则是失败的。
自从学会 `docker-compose` 的使用之后，我在本地的开发环境架构就是以下这个样子的：
![图 4](http://image.chance.fyi/image-2022082812483643888.png)
以上就是一个 `docker-compose` 搭建出来的类似于 phpstudy 一样的集成环境，所有项目共用一套环境。我也设置了多个 PHP 的版本的服务，可以根据配置文件来进行 PHP 版本的切换。但是每个项目所需的环境、扩展、服务是不尽相同的，共同使用一套环境的话，为了兼容其他环境可能会安装一些不必要的东西。所以后来我在本地的开发环境架构就变成了这个样子：
![图 5](http://image.chance.fyi/image-2022082813142891152.png)
每个项目都有一个自己的 `docker-compose` ，每个项目根据自身所需配置只属于自己的环境，不需要的服务就可以移除掉。这种环境架构感觉挺好的，到目前为止我一直是这么使用的，但是当我需要上线第二个项目时遇到了一些问题。每个项目都有依赖 Nginx、MySQL、Redis 这些服务，难道每一个项目都要起一个吗？MySQL、Redis 服务器够好的话，每个项目都起一个也没有太大的问题，但是 Nginx 需要监听 80、443 端口来绑定域名的，也就是说 Nginx 需要把 80 与 443 端口映射出来，可是两个容器是不能映射同一个宿主机端口的。
一开始我准备把 Nginx 放到宿主机上，然后将各个服务的端口映射到宿主机，然后使用 Nginx 将请求转发到各个端口上去。但是我感觉这样并不是一个很好的方案，我就想到了能不能把这些基础的服务放到一个 `docker-compose` 中，其他所有项目共用这些基础服务呢，就是下面这种架构：
![图 6](http://image.chance.fyi/image-2022082816392845326.png)
在这个架构下我们的项目就没有办法通过服务名来访问 Nginx、MySQL、Redis 等服务了。
![图 7](http://image.chance.fyi/image-2022090518055541659.png)
如上图所示，我们在一个项目里访问另一个项目中的基础服务 MySQL 时访问失败了。因为它们两个不在一个 `docker-compose` 中，没有 `network` 将其链接到一起。所以想要访问也很简单，只需要将其链接到同一个 `network` 中就可以了。
如下图所示可以在我们的项目中引入一个外部的由基础服务所创建的默认的 `basic_services_default` network ，然后将我们的项目链接到这个 network 中，再次访问 MySQL 就可以成功访问了。
![图 8](http://image.chance.fyi/image-2022090609164086333.png)
