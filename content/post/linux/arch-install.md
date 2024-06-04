---
title: "Arch Linux安装"
date: 2023-09-18T15:15:08+08:00
draft: false
categories: ["Linux"]
tags: ["Linux", "Arch"]
---

前面把开发环境切换到 WSL 了，但是在使用过程中遇到一个无法解决的 Bug [microsoft/WSL#5118](https://github.com/microsoft/WSL/issues/5118)，因为我使用 symlink 也是挺频繁的，无法忍受这个 Bug，所以决定将开发环境迁移到虚拟机 😂，经过一番了解决定使用 Arch Linux，下面是虚拟机中安装 Arch Linux 的过程。

### 检查网络

```bash
ping baidu.com
```

### 检查时间是否正确

```bash
timedatectl
```

### 创建硬盘分区

```bash
# 用 sgdisk 将 MBR 分区表转换为 GPT
sgdisk -g /dev/sda

# 列出磁盘分区 找到要分区的磁盘
fdisk -l
```

[分区方案 - Arch Linux 中文维基](https://wiki.archlinuxcn.org/wiki/%E5%88%86%E5%8C%BA#GUID_%E5%88%86%E5%8C%BA%E8%A1%A8:~:text=%E8%8E%B7%E5%8F%96%E6%9B%B4%E5%A4%9A%E4%BF%A1%E6%81%AF%E3%80%82-,%E5%88%86%E5%8C%BA%E6%96%B9%E6%A1%88,-%5B%E7%BC%96%E8%BE%91%20%7C)

#### /boot

- [UEFI](https://wiki.archlinuxcn.org/wiki/Unified_Extensible_Firmware_Interface "Unified Extensible Firmware Interface") 系统需要有 [EFI 系统分区](https://wiki.archlinuxcn.org/wiki/EFI_system_partition "EFI system partition")。

```bash
# 分区
fdisk /dev/sda
# 创建一个新分区
Command (m for help):n
Partition number (1-128,default 1):
First sector (34-209715166,default 2048):
Last sector,+/-sectors or +/-sizefK,M,G,T,P}(2048-209715166,default 209713151):+512m
Created a new partition 1 of type 'Linux filesystem'and of size 512 MiB.
# 修改分区类型为 EFI
Command (m for help): t
Partition type or alias (type L to list all):1
Changed type of partition 'Linux filesystem'to 'EFI System'.
```

#### Swap

```bash
Command (m for help):n
Partition number (2-128,default 2):
First sector(1050624-209715166,defau1t1050624):
Last sector,+/-sectors or +/-sizefK,M,G,T,P}(1050624-209715166,default 209713151):+512M
Created a new partition 2 of type 'Linux filesystem'and of size 512 MiB
Command (m for help):t
Partition number (1,2,default 2):
Partition type or alias (type L to list all):19
Changed type of partition 'Linux filesystem'to 'Linux swap'.
```

#### 根分区

```bash
Command (m for help):n
Partition number (3-128,default 3):
First sector(2099200-209715166,defau1t2099200):
Last sector,+/-sectors or +/-sizeK,M,G,T,P}(2099200-209715166,default 209713151):+44G
Created a new partition 3 of type'Linux filesystem'and of size 44 GiB.
```

#### /home

```bash
Command (m for help):n
Partition number (4-128,default 4):
First sector(94373888-209715166,defau1t94373888):
Last sector,+/-sectors or +/-sizefK,M,G,T,P}(94373888-209715166,default 209713151):
Created a new partition 4 of type 'Linux filesystem'and of size 55 GiB.
```

### 挂载分区

```bash
# 查看创建的分区
fdisk -1 /dev/sda

Disk /dev/sda:100 GiB,107374182400 bytes,209715200 sectors
Disk model:VMware Virtual S
Units:sectors of 1 512512 bytes
Sector size (logical/physical):512 bytes 512 bytes
I/0 size (minimum/optimal):512 bytes 512 bytes
Disklabel type:gpt
Disk identifier:52A3723F-2C9F-400A-B70F-E65393F881C0
Device      Start       End   Sectors Size Type
dev/sda1     2048   1050623   1048576 512M EFI System
dev/sda2  1050624   2099199   1048576 512M Linux swap
dev/sda3  2099200  94373887  92274688  44G Linux filesystem
dev/sda4 94373888 209713151 115339264  55G Linux filesystem

# 格式化分区
mkfs.ext4 /dev/sda3
mkfs.ext4 /dev/sda4
mkswap /dev/sda2
mkfs.fat -F 32 /dev/sda1

# 挂载分区
mount /dev/sda3 /mnt/
mount --mkdir /dev/sda1 /mnt/boot
mount --mkdir /dev/sda4 /mnt/home
swapon /dev/sda2
```

### 切换软件源

编辑 `/etc/pacman.d/mirrorlist`，在文件的最顶端添加：

```ini
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
```

### 安装必需的软件包

```bash
pacstrap -K /mnt base linux linux-firmware
# 一个有线所需 一个编辑器 一个补全工具
pacstrap -K /mnt dhcpcd vim bash-completion
```

### 生成 fstab 文件

通过以下命令生成 [fstab](https://wiki.archlinuxcn.org/wiki/Fstab "Fstab") 文件 (用 `-U` 或 `-L` 选项设置 UUID 或卷标)：

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

### chroot 到新安装的系统

通过以下命令 [chroot](https://wiki.archlinuxcn.org/wiki/Change_root "Change root") 到新安装的系统：

```bash
arch-chroot /mnt
```

### 设置时区

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
```

### 区域和本地化设置

程序和库如果需要本地化文本，都依赖[区域设置](https://wiki.archlinuxcn.org/wiki/Locale "Locale")，后者明确规定了地域、货币、时区日期的格式、字符排列方式和其他本地化标准。

需要设置这两个文件：`locale.gen` 与 `locale.conf`。

编辑 `/etc/locale.gen`，然后取消掉 `en_US.UTF-8 UTF-8` 和其他需要的[区域设置](https://wiki.archlinuxcn.org/wiki/Locale "Locale")前的注释（#）。

接着执行 `locale-gen` 以生成 locale 信息：

# locale-gen

然后创建 [locale.conf(5)](https://man.archlinux.org/man/locale.conf.5) 文件，并 [编辑设定 LANG 变量](https://wiki.archlinuxcn.org/wiki/Locale#%E7%B3%BB%E7%BB%9F%E5%8C%BA%E5%9F%9F%E8%AE%BE%E7%BD%AE "Locale")，比如：

创建编辑 `/etc/locale.conf`

```bash
LANG=en_US.UTF-8
```

### 网络配置

```bash
vim /etc/hostname
# 写入
archlinux

vim /etc/hosts
# 写入
127.0.0.1        localhost
::1              localhost
127.0.1.1        archlinux.localdomain        archlinux
```

### 设置 root 密码

```bash
passwd
```

### 安装引导程序

```bash
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
# 如果报错 在检查一下 分区挂载 `df -h` 没挂载重新挂载一下
# 生成主配置文件
grub-mkconfig -o /boot/grub/grub.cfg
```

### 重启

```bash
# 退出 chroot 环境
exit
# 重启
reboot
```
