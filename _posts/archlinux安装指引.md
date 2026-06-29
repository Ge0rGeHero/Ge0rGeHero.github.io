---
title: archlinux安装指引
date: 2025-10-12 11:22:29
tags:
  - linux
description: 由于本人多次安装archlinux，但经常忘记具体步骤和注意事项，导致又要重新查找资料，故写下这篇指引，将自己遇到过的问题或容易忘的事项记录下来，以便今后查阅。
---
由于本人多次安装archlinux，但经常忘记具体步骤和注意事项，导致又要重新查找资料，故写下这篇指引，将自己遇到过的问题或容易忘的事项记录下来，以便今后查阅。  
安装过程具体分为以下步骤：
1. 确认引导模式
2. 检查网络连接
3. \*更新系统时间
4. 更换仓库源
5. \*分区与格式化
6. 挂载
7. 安装系统
8. 配置系统
9. 初始化系统
# 1. 确认引导模式
通过以下命令确认引导模式：
`cat /sys/firmware/efi/fw_platform_size`
- 如果命令结果为 **64**，则系统是以 UEFI 模式引导且使用 64 位 x64 UEFI。
- 如果命令结果为 **32**，则系统是以 UEFI 模式引导且使用 32 位 IA32 UEFI，虽然其受支持，但引导加载程序只能使用 [systemd-boot](https://wiki.archlinuxcn.org/wiki/Systemd-boot "Systemd-boot") 和 [GRUB](https://wiki.archlinuxcn.org/wiki/GRUB "GRUB")。
- 如果命令结果为**No such file or directory**，则系统可能是以 [BIOS](https://zh.wikipedia.org/wiki/BIOS "zhwp:BIOS") 模式（或 [CSM](https://en.wikipedia.org/wiki/Compatibility_Support_Module "wikipedia:Compatibility Support Module") 模式，这两种模式通常出现在**老旧**的电脑或**未经配置的虚拟机**上）引导。
本教程只讲解UEFI模式的安装过程。
{% note info simple %} 提示
{% endnote %}
>使用**VMware Workstation**的用户，则需要打开虚拟机配置目录，修改用记事本或其他代码编辑器打开**archlinux.vmx**文件，并添加一行`firmware = "efi"`以确保虚拟机引导模式为**UEFI**。
# 2. 检查网络连接
用**ping**命令检查网络是否连通。
# 3. 更新系统时间
```
timedatectl set-ntp true # 将系统时间与网络时间进行同步
timedatectl status # 检查服务状态
```
# 4. 更新仓库源
- 通过curl下载中国大陆的镜像站：  
  curl -L 'https://archlinux.org/mirrorlist/?country=CN&protocol=https' -o /etc/pacman.d/mirrorlist
- 或手动编辑**/etc/pacman.d/mirrorlist**：  
  `vim /etc/pacman.d/mirrorlist`
  输入以下地址：  
  ```
  Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch # 中国科学技术大学开源镜像站
  Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch # 清华大学开源软件镜像站
  Server = https://repo.huaweicloud.com/archlinux/$repo/os/$arch # 华为开源镜像站
  Server = http://mirror.lzu.edu.cn/archlinux/$repo/os/$arch # 兰州大学开源镜像站 
  ```
# 5. 分区与格式化
{% note warning simple %} 注意
{% endnote %}
>对硬盘进行分区与格式化会对原有资料造成不可逆的效果，操作时要万分小心。
## 5.1 分区
使用`fdisk -l`检查当前硬盘情况。  
使用fdisk、cfdisk等工具修改分区表，如：
`fdisk /dev/要被分区的硬盘`  
根据[archlinux官网](https://wiki.archlinuxcn.org/wiki/%E5%AE%89%E8%A3%85%E6%8C%87%E5%8D%97)安装指引，这里建议：  
- EFI至少1GB大小
- SWAP大小与内存大小相等
## 5.2 格式化
对各分区进行格式化：
```
mkfs.ext4 /dev/_root_partition（根分区）
mkfs.fat -F 32 /dev/_efi_system_partition（EFI 系统分区）
mkswap /dev/_swap_partition（交换空间分区）
```
# 6. 挂载
遵循以下循序挂载分区：
```
mount /dev/_root_partition（根分区） /mnt # 挂载 / 目录
mkdir -p /mnt/boot # 创建 /boot 目录
mount /dev/_efi_system_partition /mnt/boot # 挂载 /boot 目录
swapon /dev/_swap_partition（交换空间分区） # 挂载交换分区
```
使用`df -h`和`free -h`检查挂载情况
# 7. 安装系统
必备软件包：`pacstrap /mnt base base-devel linux linux-firmware btrfs-progs sudo ` 
如果使用btrfs文件系统，额外安装一个btrfs-progs包。  
其他工具则根据自己的喜好进行选择，这里我列举我常用的软件：
- 文本编辑器：vim
- 联网程序：networkmanager
- 终端：zsh
- CPU微码：amd-ucode/intel-ucode，这个根据自己的CPU选择
{% note warning simple %} 注意
{% endnote %}
>执行此步骤时可能会出现文件签名损坏的提示，此时，需要更新密钥环解决，通过执行以下代码：
>```
>pacman -Scc #清除缓存
>pacman-key --init #初始化
>pacman-key --populate #导入密钥
>pacman -Sy archlinux-keyring #安装密钥环
>```
# 8. 配置系统
## 8.1 生成fstab文件
运行`genfstab -U /mnt > /mnt/etc/fstab`生成文件。  
`cat /mnt/etc/fstab`检查文件是否正确
## 8.2 登录新系统
通过**chroot**登录到新系统：  
`arch-chroot /mnt`
## 8.3 设置时间和时区
```
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime #将时区设置为北京时区
hwclock --systohc #生成/etc/adjtime
```
## 8.4 本地化设置
编辑**/etc/locale.gen**文件，删除**en_US.UTF-8**和**zh_CN.UTF-8**前的注释，然后运行：  
```
locale-gen #生成locale信息
echo 'LANG=en_US.UTF-8' > /etc/loacle.conf #创建locale.conf文件
```
## 8.5 网络配置
```
echo 'myARCH' > /etc/hostname #生成hostname文件
echo '127.0.1.1     myARCH.localdomain myARCH' > /etc/hosts  #编辑hosts文件
```
## 8.6 配置用户
### 8.6.1 设置**root**用户密码
运行：`passwd`
### 8.6.2 创建**普通**用户
```
useradd -m -G wheel -s /bin/bash 用户名
-m：在/home/目录下为该用户创建一个主目录。
-G wheel：将用户添加到wheel组，这是Arch Linux中通常用来授予sudo权限的组。﻿
-s /bin/bash：将用户默认的登录shell设置为Bash
用户名：自定义内容
```
```
passwd 用户名 #为用户设置秘密
```
为**wheel**用户组配置管理员权限，运行`EDITOR=vim visudo`，删除`%wheel ALL=(ALL:ALL) ALL`前面的注释。
## 8.7 安装引导程序
安装相关软件：  
`pacman -S grub efibootmgr os-prober`
然后运行：  
`grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ARCH`
最后生成**grub**配置文件：  
`grub-mkconfig -o /boot/grub/grub.cfg`
## 8.8 重启系统

# 9. 初始化系统
首次运行时，登录用户，然后运行以下命令开启网络连接：  
`sudo systemctl enable networkmanager --now`  
`ping baidu.com`测试网络连接。
