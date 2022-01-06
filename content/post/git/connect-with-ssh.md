---
title: "git使用SSH连接"
date: 2022-01-06T09:08:44+08:00
draft: true
categories: []
tags: []
---

## 生成 SSH 密钥

1. 打开终端`Git Bash`。

2. 粘贴下面的文本，替换为您的 GitHub 电子邮件地址。

   ```bash
   $ ssh-keygen -t ed25519 -C "your_email@example.com"
   # 创建一个新的 SSH 密钥
   Generating public/private ed25519 key pair.
   # 输入保存密钥文件位置，按Enter，接受默认位置。
   Enter a file in which to save the key (/Users/you/.ssh/id_ed25519):
   # 输入密码
   Enter passphrase (empty for no passphrase): Chance
   # 确认密码
   Enter same passphrase again:
   ```

## 将 SSH 密钥添加到 ssh-agent

1. 启动 ssh-agent 。

   ```bash
   $ eval "$(ssh-agent -s)"
   Agent pid 2016
   ```

2. 将 SSH 私钥添加到 ssh-agent ，请将`id_ed25519`修改为自己的私钥文件的名称。

   ```bash
   $ ssh-add ~/.ssh/id_ed25519
   # 输入刚刚设置的密码
   Enter passphrase for /Users/you/.ssh/id_ed25519:
   Identity added: /Users/you/.ssh/id_ed25519 (your_email@example.com)
   ```

## 将 SSH 密钥添加到 GitHub 帐户

1. 复制 SSH 公钥，请将`id_ed25519`修改为自己的公钥名称。

   ```bash
   $ clip < ~/.ssh/id_ed25519.pub
   ```

   如果`clip`不起作用，可以找到隐藏`.ssh`文件夹，在文本编辑器中打开文件，然后将其复制到剪贴板。

2. 打开 github ，点击右上角头像，然后点击 **Settings** 。

   ![userbar-account-settings](D:\HZ\Pictures\Typora\userbar-account-settings.png)

3. 用户侧边栏点击 **SSH and GPG keys** 。

   ![settings-sidebar-ssh-keys](D:\HZ\Pictures\Typora\settings-sidebar-ssh-keys.png)

4. 点击 **New SSH key** 。

   ![image-20220106105349918](D:\HZ\Pictures\Typora\image-20220106105349918.png)

5. 测试连接

   ```bash
   $ ssh -T git@github.com
   Hi Chance-fyi! You've successfully authenticated, but GitHub does not provide shell access.
   ```

   

























