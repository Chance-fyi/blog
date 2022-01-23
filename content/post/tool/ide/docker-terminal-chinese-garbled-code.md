---
title: "IDE进入Docker容器输出中文乱码"
date: 2022-01-22T10:06:42+08:00
draft: false
categories: ["IDE"]
tags: ["IDE","PHPStorm","Terminal","Docker"]
---

如下图所示，中文字符输出为乱码。

![image-20220121172638009](https://image.chance.fyi/image-20220121172638009.png)

### 解决

如下图所示编辑 **VM Options** ，添加一行 `-Dfile.encoding=UTF-8`并重启 IDE 。

![image-20220122101145122](https://image.chance.fyi/image-20220122101145122.png)

![image-20220121172834421](https://image.chance.fyi/image-20220121172834421.png)
