---
title: "使用Hugo搭建个人博客"
date: 2021-11-20T13:44:46+08:00
draft: true
---

## 安装Go

Go的安装包下载地址为[https://golang.org/dl/](https://golang.org/dl/)Windows版下载之后一路`Next`就好了，注意将Go安装目录下的`bin`目录加入到环境变量中。可使用`go version`命令查看是否安装成功。

## 安装Hugo

#### 二进制安装（推荐：简单、快速）

到 [Hugo Releases](https://github.com/spf13/hugo/releases) 下载对应的操作系统版本的Hugo二进制文件（hugo或者hugo.exe）

Windows版将下载的hugo.exe放到Go安装目录下的`bin`目录就可以了，使用`hugo version`命令查看安装是否成功。

## 创建blog

### 创建一个Github项目

使用Github进行博客的源码管理，创建一个空的Blog项目。

![image-20211118215108380](D:\88504\Desktop\Blog\使用Hugo搭建个人博客.assets\image-20211118215108380.png)

### 生成站点

```bash
# 本地创建Blog目录并进入
# 使用Hugo快速生成站点
PS D:\Blog> hugo new site .
# 初始化git 并将站点源码推送到Github
PS D:\Blog> git init
PS D:\Blog> git add .
PS D:\Blog> git commit -m "init blog" 
PS D:\Blog> git remote add origin https://github.com/Chance-fyi/Blog.git
PS D:\Blog> git push -u origin master
```

### 创建文章

创建第一篇文章，放到 `post` 目录。

```bash
PS D:\Blog> hugo new post/first.md
```

### 安装主题

hugo是没有自带默认主题的，所以必须下载一个默认主题才能展示博客的内容。可以从[主题列表](https://themes.gohugo.io/)挑选喜欢的主题下载到`themes`目录中，然后在`config.toml`添加`theme = 'PaperMod'`。

### 运行Hugo

```bash
PS D:\Blog> hugo server --buildDrafts
```

浏览器里打开： `http://localhost:1313`
