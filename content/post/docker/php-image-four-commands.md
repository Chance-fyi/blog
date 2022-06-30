---
title: "PHP镜像中自带的几个特殊的命令"
date: 2021-11-24T15:06:12+08:00
draft: false
categories: ["Docker", "PHP"]
tags: ["PHP镜像"]
---

- docker-php-source
- docker-php-ext-install
- docker-php-ext-enable
- docker-php-ext-configure

### docker-php-source

此命令，实际上就是在 PHP 容器中创建一个/usr/src/php 的目录，里面放了一些自带的文件而已。我们就把它当作一个从互联网中下载下来的 PHP 扩展源码的存放目录即可。事实上，所有 PHP 扩展源码扩展存放的路径都在` /usr/src/php/ext` 里面。

**格式**：

```bash
docker-php-source extract  # 创建并初始化 `/usr/src/php`目录
docker-php-source delete   # 删除 `/usr/src/php`目录
```

### docker-php-ext-enable

这个命令，就是用来启动 **PHP 扩展** 的。

**格式：**

```bash
docker-php-ext-enable redis  # 开启Redis扩展 前提是已经下载安装过
```

### docker-php-ext-install

这个命令，是用来安装并启动**PHP 扩展**的。

**格式：**

```bash
docker-php-ext-install 源码包目录名
```

**注意点：**

- 源码包需要放在 `/usr/src/php/ext` 下
- 默认情况下，PHP 容器没有 `/usr/src/php`这个目录，需要使用` docker-php-source extract`来生成。
- `docker-php-ext-install` 安装的扩展在安装完成后，会自动调用`docker-php-ext-enable`来启动安装的扩展。
- 卸载扩展，直接删除`/usr/local/etc/php/conf.d` 对应的配置文件即可。

### docker-php-ext-configure

`docker-php-ext-configure` 一般都是需要跟 `docker-php-ext-install`搭配使用的。它的作用就是，当你安装扩展的时候，需要自定义配置时，就可以使用它来帮你做到。

**用法：**

```bash
docker-php-ext-configure ext-name [configure flags]
```
