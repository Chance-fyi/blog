---
title: "在PhpStorm中使用PHPUnit进行单元测试"
date: 2021-12-19T11:34:23+08:00
draft: false
categories: ["PHP"]
tags: ["PHP","PHPUnit","单元测试","PhpStorm"]
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
+ 在tests目录创建StackTest.php文件，使用官网的一个例子来测试。

**StackTest.php**

```php
<?php

    namespace Chance\Log\Test;

use PHPUnit\Framework\TestCase;

class StackTest extends TestCase
{
    public function testPushAndPop()
    {
        $stack = [];
        // 断言方法 assertEquals 判断两值是否相等
        $this->assertEquals(0, count($stack));

        array_push($stack, 'foo');
        $this->assertEquals('foo', $stack[count($stack)-1]);
        $this->assertEquals(1, count($stack));

        $this->assertEquals('foo', array_pop($stack));
        $this->assertEquals(0, count($stack));
    }
}
```

## 命令行执行单元测试

运行`./vendor/phpunit/phpunit/phpunit --verbose --colors ./tests/`命令执行`tests`目录下所有单元测试。

```shell
root@7608e16a4e0a:/home# ./vendor/phpunit/phpunit/phpunit --verbose --colors ./tests/
PHPUnit 10.0-dev by Sebastian Bergmann and contributors.

Runtime:       PHP 8.1.0

.                                                                   1 / 1 (100%)

Time: 00:00.156, Memory: 6.00 MB

OK (1 test, 5 assertions)
```

可以看到执行成功并且没有错误，然后我们将倒数第二个断言的第一个变量字符串改成`foo1`再次执行单元测试。

```shell
root@7608e16a4e0a:/home# ./vendor/phpunit/phpunit/phpunit --verbose --colors ./tests/
PHPUnit 10.0-dev by Sebastian Bergmann and contributors.

Runtime:       PHP 8.1.0

F                                                                   1 / 1 (100%)

Time: 00:00.266, Memory: 6.00 MB

There was 1 failure:

1) Chance\Log\Test\StackTest::testPushAndPop
Failed asserting that two strings are equal.
--- Expected
+++ Actual
@@ @@
-'foo1'
+'foo'

/home/tests/StackTest.php:23

FAILURES!
Tests: 1, Assertions: 4, Failures: 1.
```

可以看到断言验证失败，单元测试未通过。

更多的命令选项与断言方法可自行查看文档。

## PhpStorm执行单元测试

### 配置

打开`File`->`Settings`->`PHP`->`Test Frameworks`根据自己的安装方法配置PHPUnit的执行文件地址。

![image-20211220183743437](https://image.chance.fyi/image-20211220183743437.png)

然后打开`Run`->`Edit Confiqurations`新建一个PHPUnit。

![image-20211220184707328](https://image.chance.fyi/image-20211220184707328.png)

![image-20211220190515032](https://image.chance.fyi/image-20211220190515032.png)

### 执行

![image-20211220190555597](https://image.chance.fyi/image-20211220190555597.png)

执行之后发现报错了，报错信息为`Message:  assert($testSuite instanceof TestSuiteForTestMethodWithDataProvider)`。

![image-20211220190705856](https://image.chance.fyi/image-20211220190705856.png)

经过排查发现是因为PhpStorm默认加上了`--teamcity`参数的原因，在命令行执行加这个参数也是报错，那就只有去掉这个参数了，可是PhpStorm里也没有找到怎么去掉这个默认参数。

因为我们也不需要这个参数，所以最后解决办法为，将`/home/vendor/phpunit/phpunit/src/Logging/TeamCity/TeamCityLogger.php`文件的 92 - 100 行给注释掉，就可以执行成功了。

































