---
title: "Docker中使用MySQL踩坑"
date: 2022-04-08T11:26:24+08:00
draft: true
categories: ["Docker"]
tags: ["Docker","MySQL"]
---

需要使用到 MySQL ，打算使用 Docker 来部署，compose 来管理。原以为很简单，没想到还是有些坑的。

```yaml
version: "3.7"
services:
  mysql:
    image: mysql:8.0-oracle
    container_name: mysql
    ports:
      - "3306:3306"
    restart: always
    environment:
      # 密码
      MYSQL_ROOT_PASSWORD: "root"
      # 创建默认数据库名
      MYSQL_DATABASE: "test"
```

上面的写法就可以启动一个 MySQL 服务了，很简单。但是如果重新构建的话，数据就会丢失了，解决办法也很简单，挂载一下数据目录。

```yaml
version: "3.7"
services:
  mysql:
    image: mysql:8.0-oracle
    container_name: mysql
    ports:
      - "3306:3306"
    volumes:
      - ./data/mysql:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "root"
      MYSQL_DATABASE: "test"
```

很好，MySQL 的数据已经挂载出来了，但是当重启 MySQL 容器的时候发现重启不了了，报错如下。

```bash
[Note] [Entrypoint]: Entrypoint script for MySQL Server 8.0.28-1.el8 started.
chown: cannot dereference '/var/lib/mysql/mysql.sock': No such file or directory
```

根据报错信息是找不到挂载目录下的文件，猜测是权限原因引起的，加个权限试一试。

```yaml
version: "3.7"
services:
  mysql:
    image: mysql:8.0-oracle
    container_name: mysql
    ports:
      - "3306:3306"
    volumes:
      - ./data/mysql:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "root"
      MYSQL_DATABASE: "test"
    command:
      - /bin/bash
      - -c
      - |
        chmod +rw /var/lib/mysql
        mysqld
```

给挂载目录赋予读写权限之后发现重启 MySQL 成功了，但是如果我们挂载目录是空的，也就是第一次启动 MySQL 时又报错了。

```bash
[System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.28) starting as process 1
[Warning] [MY-010159] [Server] Setting lower_case_table_names=2 because file system for /var/lib/mysql/ is case insensitive
[ERROR] [MY-011011] [Server] Failed to find valid data directory.
[ERROR] [MY-010020] [Server] Data Dictionary initialization failed.
[ERROR] [MY-010119] [Server] Aborting
[System] [MY-010910] [Server] /usr/sbin/mysqld: Shutdown complete (mysqld 8.0.28)  MySQL Community Server - GPL.
```

可以看到报错信息中有一个配置字段`lower_case_table_names`，这个配置字段是 MySQL 设置大小写是否敏感的一个参数。

**参数说明**

+ `lower_case_table_names=0` 表名存储为给定的大小和比较是区分大小写的
+ `lower_case_table_names = 1` 表名存储在磁盘是小写的，但是比较的时候是不区分大小写
+ `lower_case_table_names=2` 表名存储为给定的大小写但是比较的时候是小写的
+ **unix 、linux 下 lower_case_table_names 默认值为 0 。Windows 下默认值是 1 。Mac OS X 下默认值是 2**

MySQL 8 默认设置了`lower_case_table_names`使用 Linux/Unix 初始化参数`0`。此配置只能在 MySQL 初始化时进行设置，并且也不应该在 MySQL 运行一段时间之后去更改此配置。有两种解决方案：

**方案一**

> 我们自己来初始化 MySQL 将`lower_case_table_names`设置为`2`。

此方案就要避免数据库设计中出现大写的情况。

```yaml
version: "3.7"
services:
  mysql:
    image: mysql:8.0-oracle
    container_name: mysql
    ports:
      - "3306:3306"
    volumes:
      - ./data/mysql:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: "root"
      MYSQL_DATABASE: "test"
    command:
      - /bin/bash
      - -c
      - |
        chmod +rw /var/lib/mysql
        mysqld --initialize --lower-case-table-names=2
        mysqld
```

**方案二**

> 修改宿主机的挂载目录的是否区分大小写属性。

此方案也需要手动初始化`mysqld --initialize`，不需要修改`lower_case_table_names`了。

+ 区分大小写 `fsutil.exe file setCaseSensitiveInfo ./ enable`
+ 不区分大小写 `fsutil.exe file setCaseSensitiveInfo ./ disable`
+ 查看是否区分大小写 `fsutil.exe file queryCaseSensitiveInfo ./`
+ `./`是路径地址，在Windows 上使用，如果提示**拒绝访问**，请使用管理员权限，Mac 不知道是否可以使用。

此时即使第一次部署也不会有问题了，反复重启也没有问题，就在我以为终于成功时，数据库连接不上了:joy:。因为我们自己执行了`mysqld --initialize`，所以通过`environment`设置的环境变量都失效了。看来此做法弊大于利，放弃了。反正数据库只会初始化一次，我们就先初始化之后，在将`command`修改挂载目录权限的代码加上就好了，这样以后重启也不会有什么问题，只需要第一次执行时手动修改一下就好了:joy:。

**参考：**

+ [https://stackoverflow.com/questions/64146845/mysql-not-starting-in-a-docker-container-on-macos-after-docker-update](https://stackoverflow.com/questions/64146845/mysql-not-starting-in-a-docker-container-on-macos-after-docker-update)

+ [https://github.com/docker/for-win/issues/12384](https://github.com/docker/for-win/issues/12384)
