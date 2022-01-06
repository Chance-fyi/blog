---
title: "给Git历史提交添加GPG签名"
date: 2022-01-06T15:26:51+08:00
draft: false
categories: ["Git"]
tags: ["Git"]
---

今天给commit添加了 GPG 签名，但是以前的提交记录却没有签名。

![image-20220106153223483](https://image.chance.fyi/image-20220106153223483.png)

**使用以下命令**

> 注意：以下命令会修改历史，变更 **commit id** ，其他人会不同步，慎用。

对历史所有提交增加签名：

```bash
git filter-branch -f --commit-filter 'git commit-tree -S "$@"' HEAD
```

对最后 X 次提交增加签名：

```bash
git filter-branch -f --commit-filter 'git commit-tree -S "$@"' HEAD~X..HEAD
```

对指定提交者 Email 增加签名：

```bash
git filter-branch -f --commit-filter '
if [ "$GIT_COMMITTER_EMAIL" = "885046048@qq.com" ]
then
    git commit-tree -S "$@"
fi
'
```

**推送到远程**

```bash
git push -f
```













