---
title: "hugo添加utterances🔮评论系统"
date: 2021-11-28T13:23:17+08:00
draft: true
---

## utterances🔮

基于 GitHub issues构建的轻量级评论小部件。官网地址为[https://utteranc.es/](https://utteranc.es/)。

## 准备一个公开的github项目

需要有一个公开的项目，因为当时将博客项目设置为私有了。所以先更改一下项目权限，当然也可以新建一个项目。

![image-20211128143747359](http://r34c5ua3p.hn-bkt.clouddn.com/image-20211128143747359.png)

## 安装utterances🔮

为博客项目安装[utterances](https://github.com/apps/utterances)

![image-20211128143830157](http://r34c5ua3p.hn-bkt.clouddn.com/image-20211128143830157.png)

## 添加到博客

新建`layouts\partials\comments.html`文件，写入下面代码并将`repo`更换为自己的项目名称。

```html
<script src="https://utteranc.es/client.js"
        repo="[ENTER REPO HERE]"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
```

然后在`config`配置文件中打开评论功能`comments = true`。

## 效果展示

![image-20211128150027285](http://r34c5ua3p.hn-bkt.clouddn.com/image-20211128150027285.png)

## 问题

当切换到dark主题的时候评论还是白的，体验不太好。

![image-20211128150628174](http://r34c5ua3p.hn-bkt.clouddn.com/image-20211128150628174.png)

经过查看源码，参考这个[issues](https://github.com/utterance/utterances/issues/549)实现了根据博客主题动态切换utteranc主题。

`layouts\partials\comments.html`文件修改为：

```html
<script id='utteranc' src="https://utteranc.es/client.js"
        repo="Chance-fyi/blog"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
<script>
    document.getElementById("theme-toggle").addEventListener("click", () => {
        const theme = document.body.className.includes("dark") ? 'github-light' : 'photon-dark'
        const message = {
            type: 'set-theme',
            theme: theme
        };
        const utteranc = document.querySelector('.utterances-frame');
        utteranc.contentWindow.postMessage(message, 'https://utteranc.es');
    })
</script>
```



