---
title: "从Docker-Desktop迁移到wsl2"
date: 2023-06-02T09:18:44+08:00
draft: false
categories: ["Docker"]
tags: ["Docker", "wsl"]
---

最近更新升级了最新版本的 Docker Desktop 4.20.0 ，然后发现了一个 bug [#13524](https://github.com/docker/for-win/issues/13524)，然后降级一个版本之后又发现了另一个 bug [#13477](https://github.com/docker/for-win/issues/13477)。决定寻找替代品，尝试了 Podman Desktop 之后放弃了，最终决定直接使用 wsl2。记录一下迁移以及踩坑过程。

### WSL 遇到的无法解决的问题

- [microsoft/WSL#5118](https://github.com/microsoft/WSL/issues/5118)
  导致无法使用 pnpm，只能使用 yarn、npm 替代。

### 修改源

使用中科大的软件源 [https://mirrors.ustc.edu.cn/](https://mirrors.ustc.edu.cn/)

```bash
# 备份源文件
$ sudo mv /etc/apt/sources.list /etc/apt/sources.list.backup
# 切换 root 用户
$ sudo su
# 写入中科大的源
$ echo "deb https://mirrors.ustc.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy main restricted universe multiverse

deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-security main restricted universe multiverse

deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse

deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse

## Not recommended
# deb https://mirrors.ustc.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
# deb-src https://mirrors.ustc.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse" > /etc/apt/sources.list
# 退出 root 用户
$ exit
# 更新源
$ sudo apt-get update
```

### 安装 Docker

文档 [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)

```bash
# 卸载冲突的包
$ for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
$ sudo apt-get install ca-certificates curl gnupg
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
$ echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# https://github.com/microsoft/WSL/issues/6655
# https://patrickwu.space/2021/03/09/wsl-solution-to-native-docker-daemon-not-starting/
$ sudo update-alternatives --config iptables
There are 2 choices for the alternative iptables (providing /usr/sbin/iptables).

  Selection    Path                       Priority   Status
------------------------------------------------------------
* 0            /usr/sbin/iptables-nft      20        auto mode
  1            /usr/sbin/iptables-legacy   10        manual mode
  2            /usr/sbin/iptables-nft      20        manual mode

Press <enter> to keep the current choice[*], or type selection number: 1
update-alternatives: using /usr/sbin/iptables-legacy to provide /usr/sbin/iptables (iptables) in manual mode
# 启动 Docker
$ sudo service docker start
# 将当前用户加入 docker 用户组，重启之后当前用户可使用 Docker
$ sudo usermod -aG docker $USER
# docker 容器启用 IPv6 https://docs.docker.com/config/daemon/ipv6/#use-ipv6-for-the-default-bridge-network
```

### 设置 Git

#### 设置用户名和邮箱

```bash
$ git config --global user.name Chance
$ git config --global user.email chance.fyii@gmail.com
```

#### 设置 ssh 密钥

[生成 ssh 密钥并添加到 GitHub](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

执行 `ssh -T git@github.com` 失败

```bash
Received disconnect from 20.205.243.166 port 22:11: Bye Bye
Disconnected from 20.205.243.166 port 22
```

解决办法 [https://github.com/orgs/community/discussions/24206](https://github.com/orgs/community/discussions/24206)

```bash
# 创建 ~/.ssh/config 文件，添加以下内容
Host github.com
KexAlgorithms=ecdh-sha2-nistp521
```

#### 设置 GPG 签名

```bash
# 导入 GPG 签名
$ gpg --import GPG.asc
# 查看 GPG 签名
$ gpg --list-secret-keys --keyid-format=long
# 配置 GPG 签名
$ git config --global user.signingkey 3AA5C34371567BD2
# 默认对所有提交签名
$ git config --global commit.gpgsign true
```

### IDEA 配置

#### Docker

在 IDEA 内直接运行 `docker-compose.yml` 文件时，IDEA 默认调用的是 `docker-compose` 命令。在高版本 Docker 中，compose 是 Docker 中的一个插件，命令使用是 `docker compose`，所以运行之后报错。

```bash
The command 'docker-compose' could not be found in this WSL 2 distro.
We recommend to activate the WSL integration in Docker Desktop settings.

For details about using Docker Desktop with WSL 2, visit:

https://docs.docker.com/go/wsl2/

`docker-compose` process finished with exit code 1
```

没办法修改 IDEA 的调用命令，所以只能给 `docker compose` 设置一个别名，使用 `nano ~/.bashrc` 设置别名之后在命令行运行成功，但是 IDEA 依然报命令不存在，应该是判断了 `/usr/bin/` 目录下是否存在 `docker-compose` 命令。所以可以自己创建一个 `docker-compose` 命令脚本，在其中调用 `docker compose`。

```bash
# 创建脚本
$ sudo vim /usr/bin/docker-compose
# 输入下面脚本保存退出
docker compose "$@"
# 添加执行权限
$ sudo chmod +x /usr/bin/docker-compose
```

Docker Desktop 更新了 `4.20.1` 版本，尝试将 Docker 安装到 wsl 的 `Ubuntu 22.04.2 LTS` 还是无法工作，删除 Docker Desktop 之后，IDE 运行 compose 文件出错。

```bash
failed to solve: php:8.1.19-cli: error getting credentials - err: docker-credential-desktop.exe resolves to executable in current directory (./docker-credential-desktop.exe), out: ``
# 解决方案
# https://forums.docker.com/t/docker-credential-desktop-exe-executable-file-not-found-in-path-using-wsl2/100225
$ cat ~/.docker/config.json
{
  "credsStore": "desktop.exe"
}
# 删除 "credsStore": "desktop.exe"
```

#### GPG

在 IDE 中使用 Git 提交代码 GPG 签名失败

```bash
error: gpg failed to sign the data
fatal: failed to write commit object
```

解决方案

```bash
# https://stackoverflow.com/questions/41052538/git-error-gpg-failed-to-sign-data
$ export GPG_TTY=$(tty)
$ echo "test" | gpg --clearsign
```

#### Git

在 IDE 中读取项目的 Git 记录以及 commit 经常有一些问题，需要多次刷新，是 wsl 的 bug，可以通过 `wsl --update` 升级 wsl。

### WSL 使用本机代理上网

```bash
# https://solidspoon.xyz/2021/02/17/配置WSL2使用Windows代理上网/
# Clash 打开 Allow LAN
# 然后在本机打开对 WSL 防火墙的入站规则
$ New-NetFirewallRule -DisplayName "WSL" -Direction Inbound -InterfaceAlias "vEthernet (WSL)" -Action Allow
# 主机 IP 保存在 /etc/resolv.conf 中
$ cat /etc/resolv.conf
# 设置代理
$ vim ~/.bashrc
# 打开 ~/.bashrc 新增下面两行
export http_proxy="socks5://${hostip}:7890"
export https_proxy="socks5://${hostip}:7890"
# 测试
$ curl -vv 'https://www.google.com'
```
