---
title: "自动部署"
date: 2021-12-02T13:26:02+08:00
draft: true
---

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

编写docker-compose.yml

```yaml
version: "3.7"
services:
  git:
    image: bitnami/git
    tty: true
    working_dir: /home
    volumes:
      - ./www:/home
  go:
    image: go
    build: 
      context: ./go
    tty: true
    working_dir: /home
    volumes:
      - ./www:/home
```

配置webhooks

![image-20211202150137678](D:\HZ\Pictures\Typora\image-20211202150137678.png)

