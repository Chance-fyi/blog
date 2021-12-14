---
title: "创建一个composer包"
date: 2021-12-13T16:45:44+08:00
draft: true
---

```bash
# 1. 执行composer init
root@7608e16a4e0a:/home# composer init

  Welcome to the Composer config generator  

This command will guide you through creating your composer.json config.
# 2. 填写包名
Package name (<vendor>/<name>) [root/home]: chance/log
# 3. 输入描述
Description []: Elegant logging of operations
# 4. 输入作者
Author [, n to skip]: chance <ctx_ya@qq.com>
# 5. 最低稳定版本 可选值：stable, RC, beta, alpha, dev 一般填dev
Minimum Stability []: dev
# 6. 输入包类型
Package Type (e.g. library, project, metapackage, composer-plugin) []: library 
# 7. 输入开源协议
License []: MIT
# 8. 设置包需要依赖的其他环境或者包 下面直接回车就行了
Would you like to define your dependencies (require) interactively [yes]? 
Search for a package: 
Would you like to define your dev dependencies (require-dev) interactively [yes]? 
Search for a package: 
Add PSR-4 autoload mapping? Maps namespace "Chance\Log" to the entered relative path. [src/, n to skip]: 

{
    "name": "chance/log",
    "description": "Elegant logging of operations",
    "type": "library",
    "license": "MIT",
    "autoload": {
        "psr-4": {
            "Chance\\Log\\": "src/"
        }
    },
    "authors": [
        {
            "name": "chance",
            "email": "ctx_ya@qq.com"
        }
    ],
    "minimum-stability": "dev",
    "require": {}
}

Do you confirm generation [yes]? 

```

参考文章：[https://blog.csdn.net/Lyne_007/article/details/109626983](https://blog.csdn.net/Lyne_007/article/details/109626983)
