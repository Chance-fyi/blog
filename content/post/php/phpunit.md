---
title: "Phpunit"
date: 2021-12-19T11:34:23+08:00
draft: true
---

## PHPUnit是什么

> PHPUnit是一个面向PHP程序员的测试框架，这是一个xUnit的体系结构的单元测试框架。

PHPUnit的官网地址为：[https://phpunit.de/](https://phpunit.de/)，中文镜像网站：[http://www.phpunit.cn/](http://www.phpunit.cn/)。

## 安装PHPUnit

PHPUnit有两种安装方式，一种是下载PHAR发行包进行全局安装，一种是使用composer来为某一个项目安装。

推荐使用composer安装，本文也是使用这种安装方式。

首先以上一篇[文章](/post/php/create-composer-package/)创建的空的composer包为基础，执行以下命令即可。

```sh
root@d63b4f236f0c:/home# composer require --dev phpunit/phpunit
```

## 编写 PHPUnit 测试

+ 首先在项目下面新建一个`tests`文件夹，用来存放单元测试文件。
+ 然后编辑composer.json文件为tests文件夹增加一个命名空间`"Chance\\Log\\Test\\": "tests/"`并执行`composer dump-autoload`更新composer的命名空间与文件夹映射关系。
