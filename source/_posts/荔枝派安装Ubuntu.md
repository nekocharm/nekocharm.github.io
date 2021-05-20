---
title: 荔枝派安装Ubuntu
author: chi
avatar: 'https://cdn.jsdelivr.net/gh/nekocharm/cdn@1.0/img/custom/avatar.png'
authorLink: https://nekocharm.github.io
authorAbout: 见贤思齐焉，见不贤而内自省也
authorDesc: 见贤思齐焉，见不贤而内自省也
categories: 技术
comments: true
date: 2021-05-19 13:37:00
tags:
keywords:
description:
photo: https://img.vim-cn.com/bd/eb08fa1c05937ec3ae0aa1b97122b4582bcfc5.png
---

主要参考官方文档[Lichee zero 文档](http://zero.lichee.pro/入门/intro_cn.html) 和 [制作荔枝派Zero开发板 TF/SD卡启动盘](https://whycan.com/t_547.html)

# 编译uboot

## 安装交叉编译器

```bash
# 下载编译器 gcc-linaro-6.3.1-2017.05-x86_64_arm-linux-gnueabihf.tar.xz
tar xvf gcc-linaro-6.3.1-2017.05-x86_64_arm-linux-gnueabihf.tar.xz
vim /etc/bash.bashrc
# PATH="$PATH:./gcc-linaro-6.3.1-2017.05-x86_64_arm-linux-gnueabihf/bin"
source /etc/bash.bashrc
arm-linux-gnueabihf-gcc -v
sudo apt-get install device-tree-compiler
```

## 下载uboot源码

git clone因为网络原因可能会失败，可以下载zip解压

```bash
git clone https://github.com/Lichee-Pi/u-boot.git -b v3s-current
#or git clone https://github.com/Lichee-Pi/u-boot.git -b v3s-spi-experimental
```

## 编译

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- LicheePi_Zero_800x480LCD_defconfig
#or make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- LicheePi_Zero480x272LCD_defconfig
#or make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- LicheePi_Zero_defconfig
make ARCH=arm menuconfig
time make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- 2>&1 | tee build.log
```

## 修改环境变量

这里我修改源文件会编译不成功，所以我选择先编译再修改环境变量

```
setenv bootargs "console=ttyS0,115200 panic=5 rootwait mtdparts=spi32766.0:1M(uboot)ro,64k(dtb)ro,6M(kernel)ro,-(rootfs) root=/dev/mmcblk0p2 earlyprintk rw"
setenv bootcmd "setenv bootm_boot_mode sec; load mmc 0:1 0x41000000 zImage;load mmc 0:1 0x41800000 sun8i-v3s-licheepi-zero-dock.dtb;bootz 0x41000000 - 0x41800000; "
saveenv
```



# 编译linux内核

## 下载源码

同样可以下载zip

```bash
git clone https://github.com/Lichee-Pi/linux.git
# 默认zero-4.10.y分支，我用的5.2分支
```

## 编译源码

```bash
make ARCH=arm licheepi_zero_defconfig
make ARCH=arm menuconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16 INSTALL_MOD_PATH=out modules
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16 INSTALL_MOD_PATH=out modules_install
```

## 通过otg共享网络

目前我也没弄太明白

```bash
make ARCH=arm menuconfig
```

按下图配置

![](1.png)

![](2.png)

# 根文件系统

## 下载armhf版的ubuntu-base

>如何下载直接百度
>
>下文默认解压完在rootfs文件夹里

## 安装工具

```bash
sudo apt-get install qemu-user-static
cp /usr/bin/qemu-arm-static rootfs/usr/bin
```

## 配置DNS

直接把主机的DNS配置复制到下载的根文件系统

```bash
cp /etc/resolv.conf rootfs/etc/resolv.conf
```

## 修改软件源

```bash
sudo vim rootfs/etc/qpt/source.list
```

```vim
# 这里用的清华源
# 下面为vim命令
:% s/ports.ubuntu.com/mirrors.tuna.tsinghua.edu.cn/g
```

## 创建挂载和解挂载脚本

### 挂载

```shell
#!/bin/bash

echo "Mounting file system"

mount -t proc /proc	/rootfs/proc
mount -t sysfs /sys /rootfs/sys
mount -o bind /dev	/rootfs/dev
mount -o bind /dev/pts /rootfs/dev/pts

echo "Change root"

chroot rootfs/
```
### 解挂载

```shell
echo "Umounting file system"

umount rootfs/proc
umount rootfs/sys
umount rootfs/dev/pts
umount rootfs/dev
```

## 挂载根文件系统

```bash
sudo chmod 777 mount.sh
sudo ./mount.sh
```

## 设置密码

```bash
passwd root
```

## 设置主机名和host

```bash
echo "licheepi"> /etc/hostname
echo "127.0.0.1 localhost" > /etc/hosts
```

## 添加用户

```bash
useradd -s '/bin/bash' -m -G adm,sudo pi
passwd pi
```

## 安装常用软件

```bash
export LC_ALL=C
apt-get update
apt-get install sudo
apt-get install language-pack-en-base
apt-get install ssh
apt-get install net-tools
apt-get install ethtool
apt-get install ifupdown
apt-get install iputils-ping
apt-get install rsyslog
apt-get install htop
apt-get install vim
chmod 0400 /etc/ssh/ssh_host_*key
echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
```


> chmod 设置SSHD的本机私钥，设置其他用户不能修改
> echo 把root写入到SSHD配置文件里

## 遇到的问题

>我在登陆时遇到了两个问题
>
>1. 不能用su命令
>2. 不能用sudo命令

解决办法

```bash
chown -R root:root /bin/su
chmod u+s /bin/su
chown root:root /usr/bin/sudo
chmod 4755 /usr/bin/sudo
```

## 配置串口调试

### **此处不太清楚**按下面就成功了

```bash
cp /usr/lib/systemd/system/serial-getty@.service /etc/systemd/system/serial-getty@ttyS0.service
```
### 修改/etc/systemd/system/serial-getty@ttyS0.service

```bash
[Service]
ExecStart=-/sbin/agetty -o '-p -- \\u' --keep-baud 115200 %I $TERM 
Type=idle
```
### 链接

```bash
ln -s /etc/systemd/system/serial-getty@ttyS0.service /etc/systemd/system/getty.target.wants/
ln -s /lib/systemd/system/getty@.service /etc/systemd/system/getty.target.wants/getty@ttyS0.
service
```

## 退出

```bash
exit
sudo ./umount.sh
```

