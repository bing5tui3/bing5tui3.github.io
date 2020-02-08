---
title: Arch Linux Installation
layout: post
date:   2019-08-11
author: "bing5tui3"
tags: [linux]
categories: [linux]
excerpt_separator: <!--more-->
---

由于最近打算将局域网的测试虚拟机迁移到另外一台电脑上，并且改用**`ArchLinux`**作为测试环境操作系统，所以以本文记录如何安装**`ArchLinux`**，以供后续指导。

<!--more-->

#### 启动引导
加载镜像后，选择`Boot Arch Linux(x86_64)`
进入Arch Linux的临时版本。
由于国内连境外网络速度比较尴尬，我这边有一个局域网的梯子，所以首先设置**`HTTP`**代理（非必需的步骤）：

```bash
export http_proxy="http://10.0.1.15:8118"
```
尝试curl谷歌：
```bash
curl www.google.com.hk
```
收到返回表示代理OK！

#### 分区

```bash
cfdisk
```

选择`dos`模式。
1. 从`Free space` new 10G 出来，分区类型Primary（主分区），选择`Bootable`；
2. 从`Free space` new 2G 出来，分区类型Primary（主分区）；
3. 从`Free space` new 8G 出来，分区类型Primary（主分区）；
4. 从`Free space` new 10G 出来，分区类型Extended（扩展分区）；
5. 从拓展分区的`Free space` new 10G 出来，分区类型Primary（主分区）
选择`Write`，写回分区信息。选择`Quit`退出分区设置画面。

#### 格式化

`/dev/sda2`：挂载于根目录，格式化为`ext4`文件类型。
`/dev/sda2`：用于`swap`。
`/dev/sda3`：挂载于`/var`，格式化为`ext4`文件类型。
`/dev/sda5`：可拓展分区，挂载于`/home`，格式化为`ext4`文件类型

```bash
mkfs.ext4 /dev/sda1
mkfs.ext4 /dev/sda3
mkfs.ext4 /dev/sda5
```

设置交换分区：

```bash
mkswap /dev/sda2
swapon /dev/sda2
```

#### 挂载

将`/dev/sda1`挂载到`/mnt`下：

```bash
mount /dev/sda1 /mnt
```

创建`home` `var`目录，并挂载：

```bash
mkdir /mnt/home
mkdir /mnt/var
mount /dev/sda3 /mnt/var
mount /dev/sda5 /mnt/home
```

#### 安装 Arch Linux

引导系统安装：

```bash
pacstrap /mnt base base-devel
```

安装完成后，创建fstab文件：

```bash
genfstab /mnt >> /mnt/etc/fstab
```

更改Arch Linux的安装目录
```bash
arch-chroot /mnt /bin/bash
```

设置语言

```bash
nano /etc/locale.gen
```

取消`en_US.UTF-8 UTF-8`前面的`#`注释，保存退出。

激活语言：

```bash
locale-gen
```

将语言添加到系统中：

```bash
nano /etc/locale.conf
```

将`LANG=en_US.UTF-8`写入配置文件内，保存退出。

关联区域信息：

```bash
rm -f /etc/localtime
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

同步硬件时钟：

```bash
hwclock --systohc
```

设置主机名称：

```bash
echo 'yourHostName' > /etc/hostname
```

启动dhcp：
```bash
systemctl enable dhcpcd
```

#### 安装Bootloader

安装grub作为系统启动引擎

```bash
pacman -S grub os-prober
```

让`/dev/sda`作为boot点，将grub装入`/dev/sda`分区的起始位置：

```bash
grub-install /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```

#### 退出chroot模式，重启

```sh
exit
shutdown -r now
```

再次启动选择`Boot existing OS`即可进入Arch Linux。
