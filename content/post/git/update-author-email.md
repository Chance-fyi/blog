---
title: "修改git历史提交用户名与邮箱"
date: 2021-12-15T20:33:01+08:00
draft: false
categories: ["git"]
---

今天意外发现github上的commit记录有两个用户名，一个是公司的配置，一个是家里的配置。所以需要将已经提交的commit记录的`Author`和`email`修改回来。

## 修改本机全局用户名与邮箱

先将本机的用户名与邮箱修改正确，防止以后再次提交错误。

```bash
git config --global user.name "输入你的用户名"
git config --global user.email "输入你的邮箱"
```

## 查看历史提交信息

首先`git clone`一份代码，执行`git log`查看历史提交信息，记住要修改的`Author`和`email`。

## 批量修改历史信息

将下列代码复制到文本中修改为自己的信息，然后将代码复制到命令行回车运行。

```bash
git filter-branch -f --env-filter '
OLD_EMAIL="原来的邮箱"
CORRECT_NAME="现在的名字"
CORRECT_EMAIL="现在的邮箱"
if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_COMMITTER_NAME="$CORRECT_NAME"
    export GIT_COMMITTER_EMAIL="$CORRECT_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_AUTHOR_NAME="$CORRECT_NAME"
    export GIT_AUTHOR_EMAIL="$CORRECT_EMAIL"
fi
' --tag-name-filter cat -- --branches --tags
```

运行之后会出现一个警告，不用管等待执行结束就好了。

```bash
# 警告信息
WARNING: git-filter-branch has a glut of gotchas generating mangled history
         rewrites.  Hit Ctrl-C before proceeding to abort, then use an
         alternative filtering tool such as 'git filter-repo'
         (https://github.com/newren/git-filter-repo/) instead.  See the
         filter-branch manual page for more details; to squelch this warning,
         set FILTER_BRANCH_SQUELCH_WARNING=1.
```

然后可以再次执行`git log`可以看到已经修改成功。

## 将修改结果推送到远程

```bash
git push origin --force --all
```

