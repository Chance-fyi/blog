---
title: "Arch Linux配置"
date: 2023-09-19T14:48:41+08:00
draft: false
categories: ["Linux"]
tags: ["Linux", "Arch"]
---

### 检查网络

```bash
# 启动dhcp
systemctl start dhcpcd
# 设置开机启动
systemctl enable dhcpcd
ping baidu.com

# 检查设置系统时间
timedatectl
timedatectl set-time "yyyy-MM-dd hh:mm:ss"
# 升级系统中所有已安装的软件包
pacman -Syu
```

### 用户和用户组

```bash
# 创建用户
useradd -m chance
# 设置密码
passwd chance

# 安装sudo
pacman -S sudo
EDITOR=vim visudo
# 找到下面一行取消注释
#%wheel ALL=(ALL:ALL) ALL
# 将用户加入wheel组
gpasswd -a [用户名] [组名]
```

### SSH

```bash
pacman -S openssh
systemctl start sshd
systemctl enable sshd

# 查看ip
pacman -S net-tools
ifconfig
```

### 配置共享文件夹

```bash
pacman -S open-vm-tools
# 虚拟机设置里设置共享文件夹 名称为 share
# 查看设置的共享文件夹名字 为 share
vmware-hgfsclient
# 创建要挂载的共享文件夹
mkdir /mnt/hgfs
# 挂载
vmhgfs-fuse .host:/share /mnt/hgfs
```

#### 设置开机自动挂载

```bash
vim /etc/systemd/system/mnt_hgfs-share.service
# 设置以下内容
[Unit]
Description=Load VMware shared folders
Requires=vmware-vmblock-fuse.service
After=vmware-vmblock-fuse.service
ConditionPathExists=.host:/share
ConditionVirtualization=vmware

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/vmhgfs-fuse -o allow_other -o auto_unmount .host:/share /mnt/hgfs

[Install]
WantedBy=multi-user.target

# 运行
systemctl daemon-reload
systemctl enable mnt_hgfs-share
```

### 配置 Samba 共享文件

原本是使用的上面那种共享文件夹的方式，但是那种方式共享的文件夹文件系统也是不支持 symlink，所以切换成了这种方式，**注意不要随便恢复虚拟机的快照**，因为这种方式文件是在虚拟机内部的。

```bash
pacman -S samba
vim /etc/samba/smb.conf
# 配置
[global]
   workgroup = WORKGROUP
   server string = Samba Server
   server role = standalone server
   log file = /var/log/samba/%m.log
   max log size = 50
   dns proxy = no
[public]
  path = /home/chance/share
  public = yes
  valid users = chance
  writable = yes

# 添加用户 可以使用已有账户或创建新用户
smbpasswd -a [用户名]
chmod a+x /home/chance
mkdir /home/chance/share

systemctl start smb.service
systemctl enable smb.service
# Windows 文件管理器地址栏输入 \\ip
```

### 窗口分辨率自动适配

```bash
pacman -S gtkmm gtk2

vim /etc/mkinitcpio.conf
# 编辑 MODULES
MODULES=(vsock vmw_vsock_vmci_transport vmw_balloon vmw_vmci vmwgfx)

# 运行
mkinitcpio -p linux
systemctI enable vmtoolsd
# 重启
```

### 在没有按着 SHIFT 键时隐藏 GRUB 界面

为了获取更快的启动速度，而不用等 GRUB 倒计时，可以命令 GRUB 在启动时隐藏目录，仅在  `Shift`  被按住的时候才显示。

将如下行添加到`/etc/default/grub`来启动这个功能：

```bash
GRUB_FORCE_HIDDEN_MENU="true"
```

然后创建`/etc/grub.d/31_hold_shift`  文件并写入以下链接中的内容：[1](https://gist.githubusercontent.com/anonymous/8eb2019db2e278ba99be/raw/257f15100fd46aeeb8e33a7629b209d0a14b9975/gistfile1.sh)，给它可执行权限，然后重新生成主配置文件：

```bash
chmod a+x /etc/grub.d/31_hold_shift
grub-mkconfig -o /boot/grub/grub.cfg
# 如果脚本是从Windows复制过来的
# 可能会因为脚本文件的换行符格式不正确导致报错
# 在Linux系统中通常使用LF(\n)作为换行符 而在Windows系统中使用CRLF(\r\n)作为换行符
sed -i 's/\r//' /etc/grub.d/31_hold_shift
```

### docker

```bash
pacman -S docker
systemctI start docker
systemctI enable docker
groupadd docker
gpasswd -a [用户名] docker
```

### 安装 Clash

```bash
pacman -S clash

# 创建 systemd 配置文件 加入以下配置
vim /etc/systemd/system/clash.service

[Unit]
Description=Clash 守护进程, Go 语言实现的基于规则的代理.
After=network-online.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/bin/clash -d /etc/clash

[Install]
WantedBy=multi-user.target

# 重新加载 systemd
systemctl daemon-reload
# 启动
systemctl start clash
# 从本机的clash复制配置到共享文件夹 然后追加到现有配置 删除原有配置第一行
cat /mnt/hgfs/config.yml >> /etc/clash/config.yaml
# 重启
systemctl restart clash
# 查看状态
systemctl status clash
# 开机自启
systemctl enable clash
# 设置代理
vim /etc/environment
# 在 /etc/environment 中加入如下内容
http_proxy=127.0.0.1:7890
https_proxy=127.0.0.1:7890
socks_proxy=127.0.0.1:7891

# 面板
docker run -p 1234:80 -d --name yacd --rm ghcr.io/haishanh/yacd:master
```

### 安装 Yay

```bash
pacman -S base-devel git
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -sirc
```

### 配置 zsh

```bash
pacman -S zsh
# 安装 oh-my-zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
# 下载 powerlevel10k 主题
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
# 编辑 ~/.zshrc
ZSH_THEME="powerlevel10k/powerlevel10k"
```

### 安装桌面 i3wn

```bash
pacman -S i3-wm xorg-server xorg-xinit

# ~/.xinitrc 可以方便地在X服务器启动时运行依赖于X的程序并设置环境变量
cp /etc/X11/xinit/xinitrc ~/.xinitrc
# 编辑 ~/.xinitrc 最后一行注释并替换为
xscreensaver & exec i3
# 此时执行 startx 即可启动桌面
# 安装终端模拟器
pacman -S alacritty
```

#### ~~登录自动启动桌面~~

在您的  login shell  初始化文件（例如，[Bash](https://wiki.archlinuxcn.org/wiki/Bash "Bash")  的  `~/.bash_profile`  或  [Zsh](https://wiki.archlinuxcn.org/wiki/Zsh "Zsh")  的  `~/.zprofile`）中放置以下内容。因为上面设置了 Zsh ，不会执行 Bash 了，只能设置  `~/.zprofile`

```bash
if [ -z "${DISPLAY}" ] && [ "${XDG_VTNR}" -eq 1 ]; then
  exec startx
fi
```

#### 登录管理器

设置了登录管理器之后，上面的[[ArchLinux 配置#登录自动启动桌面]]就不需要设置了。

```bash
pacman -S lightdm lightdm-webkit2-greeter lightdm-webkit-theme-litarvan

vim /etc/lightdm/lightdm.conf
# 修改
greeter-session=lightdm-webkit2-greeter

vim /etc/lightdm/lightdm-webkit2-greeter.conf
# 修改 theme 或者 webkit-theme 为 litarvan

systemctl start lightdm
systemctl enable lightdm
```

#### 与宿主机共享剪贴板

```bash
pacman -S gtkmm3
# 创建的环境变量
vim ~/.xprofile

#!/bin/sh
vmware-user &
```

#### i3bar

```bash
pacman -S polybar
mkdir ~/.config/polybar
cp /etc/polybar/config.ini ~/.config/polybar/config.ini
# 编写启动脚本
vim ~/.config/polybar/launch.sh
# 写入如下内容
!/bin/bash

# 终止正在运行的 bar 实例
killall -q polybar
# 如果你所有的 bar 都启用了 ipc，你也可以使用
polybar-msg cmd quit

# 运行 Polybar，使用默认的配置文件路径 ~/.config/polybar/config.ini
polybar 2>&1 | tee -a /tmp/polybar.log & disown

echo "Polybar launched..."

# 设置执行权限
chmod +x ~/.config/polybar/launch.sh

# 关闭 i3 的 bar
vim ~/.config/i3/config
# 注释掉 bar{ } 段落
# 在最后添加启动脚本
exec_always --no-startup-id $HOME/.config/polybar/launch.sh
```

#### 字体

```bash
# 安装字体
pacman -S ttf-roboto noto-fonts noto-fonts-cjk adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts ttf-dejavu

vim ~/.config/fontconfig/fonts.conf
```

[字体配置/中文 - Arch Linux 中文维基](https://wiki.archlinuxcn.org/wiki/%E5%AD%97%E4%BD%93%E9%85%8D%E7%BD%AE/%E4%B8%AD%E6%96%87)

```bash
# 系统中文化
vim /etc/locale.gen
# 取消注释
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
zh_SG.UTF-8 UTF-8

# 执行 `locale-gen` 命令
locale-gen
vim ~/.xprofile
# 文件开头添加
export LANG=zh_CN.UTF-8
export LANGUAGE=zh_CN:en_US
```

#### 应用程序启动器

```bash
yay -S ulauncher

vim ~/.config/i3/config
# 添加开机启动
exec --no-startup-id ulauncher --hide-window
```

#### 输入法

```bash
pacman -S fcitx5-im fcitx5-chinese-addons
yay -S fcitx5-input-support

vim ~/.config/i3/config
# 添加开机启动
exec --no-startup-id fcitx5 -d

# 配置
fcitx5-configtool

# 安装词库
pacman -S fcitx5-pinyin-zhwiki

# 安装主题
pacman -S fcitx5-material-color
vim ~/.config/fcitx5/conf/classicui.conf
# 垂直候选列表
Vertical Candidate List=False
# 按屏幕 DPI 使用
PerScreenDPI=True
# Font (设置成你喜欢的字体)
Font="思源黑体 CN Medium 13"
# 主题
Theme=Material-Color-Pink
# 也可前往 `Fcitx5设置 -> 配置附加组件 -> 经典用户界面 -> 主题` 设置主题。
```

#### 壁纸

```bash
pacman -S feh
mkdir ~/.wallpaper
curl -o ~/.wallpaper/arch.jpg https://w.wallhaven.cc/full/4x/wallhaven-4x2orl.png
feh --bg-scale ~/.wallpaper/arch.jpg

vim ~/.config/i3/config
# 添加开机启动
exec --no-startup-id feh --bg-scale ~/.wallpaper/arch.jpg
```

### 软件

```bash
# 替代 cat
pacman -S bat

# 更现代的 top
pacman -S bottom

# 可更正以前控制台命令中的错误
pacman -S thefuck
```

#### zsh

```bash
pacman -S zsh
# 安装 oh-my-zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

#### zsh 插件

##### aliases

查看别名

##### fzf-tab

将 zsh 的默认完成选择菜单替换为 fzf！

```bash
# 安装 zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

# 安装 fast-syntax-highlighting
git clone https://github.com/zdharma-continuum/fast-syntax-highlighting.git \
  ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/plugins/fast-syntax-highlighting

# 安装 fzf
pacman -S fzf

# 安装 fzf-tab
git clone https://github.com/Aloxaf/fzf-tab ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/fzf-tab
```

##### sudo

连续两次 ESC 给上调命令加 sudo

##### zoxide

Zoxide 是一个更智能的 CD 命令，它会记住您最常使用的目录，因此您只需按几下键即可“跳转”到它们。

##### extract

提供命令 `x` 解压任何类型文件

##### zsh-autosuggestions

作用是根据历史输入命令的记录即时的提示（建议补全），然后按 → 键即可补全。

##### zsh-syntax-highlighting

作用：命令错误会显示红色，直到你输入正确才会变绿色，另外路径正确会显示下划线。

```bash
git clone --depth=1 https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

#### zsh 主题

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
# Set `ZSH_THEME="powerlevel10k/powerlevel10k"` in `~/.zshrc`
```
