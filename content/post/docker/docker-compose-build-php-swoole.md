---
title: "使用docker-compose构建PHP8.0 + Swoole + Redis + MongoDB环境"
date: 2021-11-24T15:36:45+08:00
draft: true
---

首先创建我们的工作目录`Docker`，然后在[DockerHub](https://hub.docker.com/)上查找PHP8.0最新版本的镜像目前为`PHP8.0.12`，在`Docker`目录下创建`php`目录，因为以后可能会使用别的版本的PHP，所以在`php`目录下在创建一个`php8.0.12`的目录，并在目录中创建`Dockerfile`文件。

目录结构为

```
Docker
├─ php
│   ├─ php8.0.12
│   │   └─ Dockerfile
│   ├─ ...
```

## 编写PHP的Dockerfile

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

# 下载pdo_sqlsrv扩展
RUN printf "" | pecl install pdo_sqlsrv-5.9.0

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
```

#### `docker-php-source extract` 创建的目录都存放了哪些扩展源码?

`docker-php-source extract`命令创建的目录其实存放了一些扩展的源码，我们直接安装就可以了。不用在使用pecl安装扩展了，可以运行一个容器查看有哪些存在的扩展。

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

想了解上面Dockerfile中出现的`docker-php-`开头的命令是什么意思，可以查看这一篇[文章](/post/docker/php-image-four-commands)。

### 优化Dockerfile

我们根据上面的Dockerfile可以制作出我们需要的镜像，但是这种写法存在很多问题，并不是最完美的。
