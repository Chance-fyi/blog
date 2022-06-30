---
title: "使用docker-compose构建PHP8.0 + Swoole + Redis + MongoDB环境"
date: 2021-11-24T15:36:45+08:00
draft: false
categories: ["Docker", "PHP"]
tags: ["PHP", "Swoole", "Docker", "docker-compose", "PHP环境搭建"]
---

最近公司考虑使用 PHP8 + swoole 进行项目的重构，所以要搭建一个环境进行开发学习。swoole 只能运行在 Linux 系统下，考虑到统一团队的开发环境，避免别的小伙伴在环境问题上浪费太多时间，所以选择使用 docker 来构建一个开发环境。

首先创建我们的工作目录`Docker`，然后在[DockerHub](https://hub.docker.com/)上查找 PHP8.0 最新版本的镜像目前为`PHP8.0.12`，在`Docker`目录下创建`php`目录，因为以后可能会使用别的版本的 PHP，所以在`php`目录下在创建一个`php8.0.12`的目录，并在目录中创建`Dockerfile`文件。

目录结构为

```
Docker
├─ php
│   ├─ php8.0.12
│   │   └─ Dockerfile
│   ├─ ...
```

## 构建自己的 PHP 镜像

### Dockerfile

```dockerfile
# 因为要使用swoole直接使用cli版本
FROM php:8.0.12-cli

# 更新依赖
RUN apt-get update \
	&& apt-get install -y \
		unixodbc-dev \
		zlib1g-dev \
		libzip-dev

# 创建`/usr/src/php/ext`目录
RUN docker-php-source extract

# 下载redis扩展
# printf "" | 是为了跳过扩展安装过程中弹出让我们选择的yes no
RUN printf "" | pecl install redis-5.3.4

# 下载mongodb扩展
RUN printf "" | pecl install mongodb-1.11.1

# 下载xlswriter扩展
RUN printf "" | pecl install xlswriter-1.5.1

# 下载swoole扩展
RUN printf "" | pecl install swoole-4.8.1

# 安装composer
RUN curl -sS https://getcomposer.org/installer | php \
    && mv composer.phar /usr/bin/composer && chmod +x /usr/bin/composer

# 安装扩展
RUN docker-php-ext-install pdo_mysql \
        zip \
        pcntl \
    # 开启扩展
    && docker-php-ext-enable redis \
        mongodb \
        pdo_sqlsrv \
        xlswriter \
        swoole
# 删除`/usr/src/php/ext`目录
RUN docker-php-source delete \
    && rm -rf /tmp/*
```

#### `docker-php-source extract` 创建的目录都存放了哪些扩展源码?

`docker-php-source extract`命令创建的目录其实存放了一些扩展的源码，我们直接安装就可以了。不用在使用 pecl 安装扩展了，可以运行一个容器查看有哪些存在的扩展。

```bash
PS D:\Docker> docker run -it php:8.0.12-cli sh
/ # docker-php-source extract
/ # ls /usr/src/php/ext/
bcmath        date          ffi           gmp           ldap          odbc          pdo_dblib     pdo_sqlite    reflection    soap          sysvmsg       xmlreader
bz2           dba           fileinfo      hash          libxml        opcache       pdo_firebird  pgsql         session       sockets       sysvsem       xmlwriter
calendar      dom           filter        iconv         mbstring      openssl       pdo_mysql     phar          shmop         sodium        sysvshm       xsl
com_dotnet    enchant       ftp           imap          mysqli        pcntl         pdo_oci       posix         simplexml     spl           tidy          zend_test
ctype         exif          gd            intl          mysqlnd       pcre          pdo_odbc      pspell        skeleton      sqlite3       tokenizer     zip
curl          ext_skel.php  gettext       json          oci8          pdo           pdo_pgsql     readline      snmp          standard      xml           zlib
```

想了解上面 Dockerfile 中出现的`docker-php-`开头的命令是什么意思，可以查看这一篇[文章](/post/docker/php-image-four-commands)。

### 优化 Dockerfile

根据上面的 Dockerfile 可以制作出需要的镜像，但是这种写法存在很多问题，并不是最完美的。了解 Docker 的应该知道，镜像是分层的，使用 Dockerfile 构建镜像时，每一条指令构建一层。为了更好的排错，所以每一个操作使用了一条指令，虽然在最后清理了所有下载、展开的文件，在运行的容器中也确实看不到这些文件了，但是这些文件其实还是存在于镜像之中，这样会使镜像非常臃肿。

**优化后的 Dockerfile**

```dockerfile
# 因为要使用swoole直接使用cli版本
FROM php:8.0.12-cli

# 更新依赖
RUN set -x \
    && apt-get update \
    && apt-get install -y unixodbc-dev zlib1g-dev libzip-dev \
    # 创建`/usr/src/php/ext`目录
    && docker-php-source extract \
    # 下载redis扩展
    # printf "" | 是为了跳过扩展安装过程中弹出让我们选择的yes no
    && printf "" | pecl install redis-5.3.4 \
    # 下载mongodb扩展
    && printf "" | pecl install mongodb-1.11.1 \
    # 下载xlswriter扩展
    && printf "" | pecl install xlswriter-1.5.1 \
    # 下载swoole扩展
    && printf "" | pecl install swoole-4.8.1 \
    # 安装composer
    && curl -sS https://getcomposer.org/installer | php \
    && mv composer.phar /usr/bin/composer && chmod +x /usr/bin/composer \
    # 安装扩展
    && docker-php-ext-install pdo_mysql \
        zip \
        pcntl \
    # 开启扩展
    && docker-php-ext-enable redis \
        mongodb \
        pdo_sqlsrv \
        xlswriter \
        swoole \
    # 删除`/usr/src/php/ext`目录
    && docker-php-source delete \
    && rm -rf /tmp/* \
    # 清理apt缓存文件
    && apt-get purge
```

## 构建`php + redis + mongo`服务

自定义的 php 镜像 Dockerfile 文件已经制作好了，接下来使用 docker-compose 来构建`php + redis + mongo`的服务。

首先在 Docker 目录下新建一个配置文件`.env`。

**.env**

```ini
# 配置使用的php版本
PHP_VERSION=8.0.12
```

然后在 Docker 目录下新建`docker-compose.yml`。

**docker-compose.yml**

```yaml
version: "3.7"
services:
	# 服务名 可以与其他服务互通
    php:
    	# 自定义构建的镜像名称
    	# ${PHP_VERSION} .env配置文件配置的版本
        image: my_php:${PHP_VERSION}
        build:
        	# 使用的Dockerfile文件路径
            context: ./php/php${PHP_VERSION}
        ports:
            - 9000:9000
            - 9501:9501
        volumes:
        	# www目录位于Docker目录下 用来存放代码
            - ./www:/home
            # 先暂时注释配置文件的目录挂载
            # 等容器启动起来之后先将配置复制出来
            # 然后在打开目录挂载
            # - ./php/php${PHP_VERSION}/conf:/usr/local/etc/
        working_dir: /home
        # php-cli版本的镜像没有前台进程 容器启动没有前台进程就会挂掉
        tty: true
    redis:
        image: redis
        ports:
            - 6379:6379
        volumes:
           # - ./redis/conf:/usr/local/etc/redis
    mongo:
        image: mongo
        restart: always
        ports:
          - 27017:27017
        environment:
            MONGO_INITDB_ROOT_USERNAME: root
            MONGO_INITDB_ROOT_PASSWORD: example
```

## 启动服务

```bash
# 启动服务
PS D:\Docker> docker-compose up -d
# 查看运行的容器
PS D:\Docker> docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED          STATUS          PORTS                                            NAMES
c7c0573ad9a9   nginx           "/docker-entrypoint.…"   45 seconds ago   Up 40 seconds   0.0.0.0:80->80/tcp, 0.0.0.0:8080->8080/tcp       docker-nginx-1
1ad5aa8339af   mongo           "docker-entrypoint.s…"   45 seconds ago   Up 43 seconds   0.0.0.0:27017->27017/tcp                         docker-mongo-1
01d2b6f56e17   my_php:8.0.12   "docker-php-entrypoi…"   45 seconds ago   Up 42 seconds   0.0.0.0:9000->9000/tcp, 0.0.0.0:9501->9501/tcp   docker-php-1
8a6fbf0bc833   redis           "docker-entrypoint.s…"   45 seconds ago   Up 41 seconds   0.0.0.0:6379->6379/tcp                           docker-redis-1
# 将PHP的配置从容器复制出来 然后可以将上面的配置挂载注释解开
PS D:\Docker> docker cp 01d2b6f56e17:/usr/local/etc/ ./php/php8.0.12/conf/
```

## 测试

在`www`目录下创建一个`swoole.php`。

**swoole.php**

```php
<?php

$http = new Swoole\Http\Server('0.0.0.0', 9501);

$http->on('Request', function ($request, $response) {
    $redis = new Redis();
    // 连接Redis不使用ip连接 容器的ip是会改变的 使用服务名来访问Redis服务
    // docker-compose会帮我们构建一个network 同一个yml下的服务会共用一个network
    // 使用MongoDB也是一样
    $redis->connect('redis', 6379);
    $redis->set('key',rand(1000, 9999));
    $val = $redis->get('key');

    $response->header('Content-Type', 'text/html; charset=utf-8');
    $response->end('Hello Swoole. #' . $val);
});

$http->start();
```

**运行测试**

```bash
PS D:\Docker> docker exec -it 01d2b6f56e17 php swoole.php
```

**运行结果**

![image-20211125163033206](https://image.chance.fyi/image-20211125163033206.png)
