---
title: "自动部署"
date: 2021-12-18T13:26:02+08:00
draft: false
categories: ["Hugo","git"]
tags: ["Hugo","git","webhook","自动部署","Docker","docker-compose"]
---

每次写一篇新的文章或者有一点改动，都要手动放到服务器上，特别麻烦，所以今天来搭建一个自动部署环境。每次修改之后只要push到github上就可以了。

我对Docker还是很喜欢的，所以这次环境也用Docker进行搭建。

## 安装Docker

安装docker

```bash
# 卸载旧版本
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
# 设置存储库
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
# 安装 Docker 
sudo yum install docker-ce docker-ce-cli containerd.io
# 启动docker
sudo systemctl start docker

# 运行此命令以下载 Docker Compose 的当前稳定版本
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# 对二进制文件应用可执行权限
sudo chmod +x /usr/local/bin/docker-compose
```

## 搭建环境

> 环境的源码可以到我的github[查看](https://github.com/Chance-fyi/cvm-docker/releases/tag/v1.0.0)

首先确认一下我们的环境需要什么东西：

+ `Git`首先需要Git从github上拉取源码
+ `hugo`然后使用hugo来将源码生成静态文件
+ `webhook`使用github的webhook来完成自动部署
+ `Nginx`使用Nginx完成域名解析与webhook工具的反向代理

webhook我工具选择使用[adnanh/*webhook*](https://github.com/adnanh/webhook)，这款工具和hugo一样是使用Go语言开发的，所以只需要下载可执行文件就可以使用了，非常方便。

### 构建镜像

我们使用Git镜像`bitnami/git`来作为基础镜像，然后下载[hugo](https://github.com/gohugoio/hugo/releases)与[webhook](https://github.com/adnanh/webhook/releases/tag/2.8.0)的可执行文件。

**Dockerfile**

```dockerfile
FROM bitnami/git

# 将hugo与webhook可执行文件复制到镜像中
COPY hugo /usr/local/bin
COPY webhook /usr/local/bin

# 添加执行权限
RUN chmod +x -R /usr/local/bin
```

### webhook

准备webhook的配置文件，查看webhook配置文件的[模板](https://github.com/adnanh/webhook/blob/master/docs/Hook-Examples.md)，以及配置文件中字段的[含义](https://github.com/adnanh/webhook/blob/master/docs/Hook-Definition.md)。

webhook启动之后默认监听`:9000/hooks/{id}`路径下的请求，id为hooks.json中配置的id。

**hooks.json**

```json
[
    {
        "id": "blog",
        "execute-command": "/sh/blog-webhook.sh",//这里试了几次好像不能直接执行命令，只能写成shell脚本
        "trigger-rule":
        {
            "and":
            [
                {
                    "match":
                    {
                        "type": "payload-hmac-sha1",
                        "secret": "你在github设置的secret",
                        "parameter":
                        {
                            "source": "header",
                            "name": "X-Hub-Signature"
                        }
                    }
                },
                {
                    "match":
                    {
                        "type": "value",
                        "value": "refs/heads/master",
                        "parameter":
                        {
                            "source": "payload",
                            "name": "ref"
                        }
                    }
                }
            ]
        }
    }
]
```

**blog-webhook.sh**

```sh
#!/bin/sh
cd "/home/blog"
git pull
hugo
```

### github

![image-20211218211108204](https://image.chance.fyi/image-20211218211108204.png)

### Nginx

上面说了webhook监听9000端口，所以配置一个域名转发到9000端口。

**webhook.conf**

```conf
server {
    listen       80;
    listen  [::]:80;
    server_name  webhook.chance.fyi;

    #把http的域名请求转成https
    return 301 https://$host$request_uri;
}
server {
    #SSL 访问端口号为 443
    listen 443 ssl;
    server_name webhook.chance.fyi;

    location / {
        proxy_pass  http://blog:9000;
        proxy_set_header Host $proxy_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

> 可能看到这个`proxy_pass  http://blog:9000;`会不太理解，
>
> 为什么不是`proxy_pass  http://127.0.0.1:9000;`呢。
>
> 因为我们下面会使用docker-compose来编排服务，一个是`nginx`服务一个是`blog`服务，两者之间是通过服务名访问。

## docker-compse编排

**docker-compose.yml**

```yaml
version: "3.7"
services:
  blog:
    image: blog
    build: ./git
    working_dir: /home
    tty: true
    ports:
      - 9000:9000
    volumes:
      - ./www:/home
      - ./webhook:/webhook
      - ./sh:/sh
    command: 
      - /bin/bash
      - -c
      - |
        rm -rf /home/blog
        cd /home
        git clone git://github.com/Chance-fyi/blog.git
        cd /home/blog
        hugo
        chmod +x -R /sh
        webhook -hooks /webhook/hooks.json -hotreload
  nginx:
    image: nginx
    working_dir: /home
    tty: true
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./www:/home
      - ./sh:/sh
      - ./nginx/conf:/etc/nginx/conf.d
      - ./nginx/cert:/etc/nginx/cert
```

## 问题

### Nginx服务退出

当我执行`docker-compose up -d`之后nginx服务立即退出了，原因是`webhook`还没有启动成功，而`webhook.conf`配置中又依赖于webhook，监听到9000端口还没启动就报错退出了。

#### 解决办法

使用一个[wait-for-it](https://github.com/vishnubob/wait-for-it)的shell脚本来监听webhook是否启动成功，然后在启动Nginx。

修改后的docker-compose.yml文件为：

```yaml
version: "3.7"
services:
  blog:
    image: blog
    build: ./git
    working_dir: /home
    tty: true
    ports:
      - 9000:9000
    volumes:
      - ./www:/home
      - ./webhook:/webhook
      - ./sh:/sh
    command: 
      - /bin/bash
      - -c
      - |
        rm -rf /home/blog
        cd /home
        git clone git://github.com/Chance-fyi/blog.git
        cd /home/blog
        hugo
        chmod +x -R /sh
        webhook -hooks /webhook/hooks.json -hotreload
  nginx:
    image: nginx
    working_dir: /home
    tty: true
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./www:/home
      - ./sh:/sh
      - ./nginx/conf:/etc/nginx/conf.d
      - ./nginx/cert:/etc/nginx/cert
    command: 
      - /bin/bash
      - -c
      - |
        chmod +x -R /sh
        /sh/wait-for-it.sh blog:9000 -- nginx -g 'daemon off;'
```

再次`docker-compose up -d`之后Nginx服务就不会退出啦。

### blog-webhook.sh执行失败

当我push到github上的时候，webhook请求是成功了的，但是修改并没有被更新。

经过排查发现如下报错：

```sh
error occurred: fork/exec /sh/blog-webhook.sh: no such file or directory
```

排查发现文件是存在的，然后进入容器执行这个脚本：

```sh
root@cc8ce34686cd:/home# /sh/blog-webhook.sh 
bash: /sh/blog-webhook.sh: /bin/bash^M: bad interpreter: No such file or directory
```

经过搜索得知是因为在Windows下编辑的sh脚本是dos格式的，而Linux环境下需要文件是unix格式才行。

#### 解决办法

参考这篇[文章](https://blog.csdn.net/CodyGuo/article/details/72811173?locationNum=1&fps=1)使用vscode修改了文件格式，再上传就没有问题了。





