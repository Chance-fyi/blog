---
title: "PhpStorm使用xdebug远程调试docker环境内的代码"
date: 2021-12-11T13:57:27+08:00
draft: true
---

## 安装 xdebug

### 1、查询 xdebug 版本

输出`phpinfo()`，然后复制源码。

然后使用 xdebug 官方提供的一个检测工具：[https://xdebug.org/wizard](https://xdebug.org/wizard)

![image-20211211133611808](D:\88504\Pictures\Typora\image-20211211133611808.png)

将`phpinfo`的源码复制进去，然后点击`Analyse my phpinfo() output`按钮。就可以获取到我们需要的`xdebug`版本和安装方式。

![image-20211211133915026](D:\88504\Pictures\Typora\image-20211211133915026.png)

可以按照上述方式安装，不过我用的是`docker`，所以就只需要`xdebug`版本就好了。

给我的`Dockerfile`文件加两行代码。

```dockerfile
FROM php:8.1.0-cli

RUN set -x \
	......
	# 下载xdebug扩展
    && printf "" | pecl install xdebug-3.1.2 \
    ......
    # 开启扩展
    && docker-php-ext-enable xdebug
    ......
```

docker 环境的话可以查看这一篇[文章](/post/docker/docker-compose-build-php-swoole)。

> 注意：我现在使用的环境是没有 swoole 扩展的，你如果使用了我前面搭建的环境，请把 swoole 扩展删掉，因为`swoole`和`xdebug`是不兼容的。
>
> 还有配置文件目录记得先复制在映射。

`php.ini`或者`docker-php-ext-xdebug.ini`文件中加入以下配置：

```ini
;开启xdebug支持远程调试
xdebug.remote_enable=1
;自动触发调试
xdebug.remote_autostart=1
xdebug.remote_handler=dbgp
xdebug.remote_mode = req
xdebug.remote_host=host.docker.internal
xdebug.remote_port=9111
xdebug.idekey = "PHPSTORM"
xdebug.remote_log=/tmp/xdebug.log
```

## 配置 PhpStorm

![image-20211211151829955](D:\88504\Pictures\Typora\image-20211211151829955.png)

![image-20211211151918880](D:\88504\Pictures\Typora\image-20211211151918880.png)

![image-20211211152042087](D:\88504\Pictures\Typora\image-20211211152042087.png)

![image-20211211152111852](D:\88504\Pictures\Typora\image-20211211152111852.png)

![image-20211211152507069](D:\88504\Pictures\Typora\image-20211211152507069.png)

![image-20211211152410270](D:\88504\Pictures\Typora\image-20211211152410270.png)
