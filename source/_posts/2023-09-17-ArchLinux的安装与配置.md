---
layout: post
title:  "Arch"
date:   2023-09-17 10:10:40 +0800
categories: 
- [blog]
- [linux]
tags:
- Arch
- Linux
---
Arch —— A simple, lightweight distribution
介绍Arch的安装与配置，应用配置待续。

<!-- more -->

## Arch Linux安装

### 安装前准备

>官方网站：[Arch Linux](https://archlinux.org/)  
>Arch Wiki：[ArchWiki (archlinux.org)](https://wiki.archlinux.org/)  
>Ventory：[Ventoy](https://www.ventoy.net/cn/)


从官网的[Arch Linux下载页面](https://archlinux.org/download/)下载镜像之后，只需要复制到已经制作好的Ventory安装U盘中即可，制作VentoryU盘的方法参考[Ventory文档手册](https://www.ventoy.net/cn/doc_start.html)。

> 通常来说，下载之后验证文件完整性和签名是必要的

针对我的设备，由于之前已经多次安装过Linux系统，也没有使用双系统的想法，所以直接执行全盘抹除安装，也很清楚系统是UEFI启动，因此不需要在BIOS中进行调整。多数情况下，也只需要禁用secure boot即可。

### 安装流程

1.  启动到live环境

    ！！！！确保当前运行的系统下无重要文件需要备份！！！！
    在VentoryU盘插入之后，重启电脑，按F2键（长按或狂按或其他键）进入BIOS模式。在BIOS模式中禁用secure boot并调整启动项顺序，将U盘启动调整至第一位，然后保存设置并退出重启。
    此次重启将默认进入Ventory启动，在图形界面中选择Arch Linux进行安装，引导选择GRUB或默认，进入Arch引导程序后，选择第一项：

    ```text
    Arch Linux install medium (x86_64, UEFI)
    ```

    就会成功进入Arch Live环境

2.  验证启动模式  
    新电脑默认UEFI，在此不再验证。

3.  连接互联网  
    有线连接通常插上网线即可，在此记录无线连接步骤。  
    输入iwctl命令，进入iwd环境  
    输入 device list 得到本机网卡设备名称（通常为wlan0，**下文中以wlan0代指**）  
    扫描WiFi：station wlan0 scan  
    输出扫描结果：station wlan0 get-networks  
    记住你要连接的网络SSID，此字段在输出结果中的Network Name中，**下文以MySSID代指**  
    连接网络：station wlan0 connect “MySSID”  
    输入密码即可，随后输入quit退出iwd环境  
    ping www.baidu.com 以测试网络连接

4.  更新系统时钟

    ```text
    timedatectl status
    ```

5.  **磁盘分区**

    > 以下分区表仅代表个人喜好，无通用性，首次尝试建议单独分区home！！

    | 挂载点 | 分区 | 分区类型 | 建议大小 | 备注 |代码|
    |-----|----|-----|------|------|----
    |/mnt/boot|/dev/nvme0n1p1|EFI|512M|ESP|
    |/mnt|/dev/nvme0n1p2|Linux x86_64 root|所有剩余空间|根分区|
    | /swap| /dev/nvme0n1p3|Swap|与内存大小相同|交换分区|

    查看分区信息：fdisk -l
    进入磁盘编辑：fdisk /dev/nvme0n1 （以nvme0n1为例）
    确保在看到如下环境提示后进行

    ```text
    root@archiso ~ # fdisk /dev/nvme0n1

    Welcome to fdisk (util-linux 2.xx).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.
    ```

    输入 g 创建新的GPT分区表

    ```text
    Command (m for help): n  # 输入 n 创建新的分区，这个分区将是 EFI 分区
    Partition number (1-128, default 1):  # 分区编号保持默认，直接按 Enter
    First sector (2048-125829086, default 2048):  # 第一个扇区，保持默认
    Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-125829086, default 125827071): +512M  # 创建 512MiB 大小的分区

    Do you want to remove the signature? [Y]es/[N]o: Y  # 清除已有的签名，如果是全新的硬盘，则没有此步

    Command (m for help): t  # 输入 t 改变分区类型，请勿遗忘此步
    Selected partition 1
    Partition type or alias (type L to list all): 1  # 输入 1 代表 EFI 类型
    Changed type of partition 'Linux filesystem' to 'EFI System'.

    Command (m for help): p  # 输入 p 打印分区表，请检查分区是否有误，如果有误，请输入 q 直接退出


    Command (m for help): w  # 输入 w 写入分区表，该操作不可恢复

    ```

6.  分区格式化
    格式化根分区：

    ```text
    mkfs.ext4 /dev/nbme0n1p2
    ```

    格式化swap分区

    ```text
    mkswap /dev/nvme0n1p3
    ```

    格式化EFI分区

    ```text
    mkfs.fat -F 32 /dev/nvm10n1p1
    ```

7.  挂载分区
    挂载根分区

    ```text
    mount /dev/nvme0n1p2 /mnt
    ```

    创建boot目录并挂载EFI分区

    ```text
    mount --mkdir /dev/nvme0n1p1 /mnt/boot
    ```

    启用swap分区

    ```text
    swapon /dev/nvme0n1p3
    ```

8. 选择镜像源
    非常重要，由于众所周知的网络环境的问题，默认的镜像源会对之后的下载和更新过程带来极大的速度影响。
    推荐使用如下命令筛选镜像源，这将为您选出位于平均同步延迟在 3 小时以内的，位于中国的 https 软件源，并根据速度排序。指定 --completion-percent 95（默认为100）的目的是防止忽略可用的镜像源。

    ```text
    root@archiso ~ # reflector -p https -c china --delay 3 --completion-percent 95 --sort rate --save /etc/pacman.d/mirrorlist
    ```

    也可以手动编辑/etc/pacman.d/mirrorlist文件来选择镜像源

9.  安装基础包

    ```text
    pacstrap -K /mnt base linux linux-firmware
    ```

10. 配置系统
    生成fstab文件

    ```text
    genfstab -U /mnt >> /mnt/etc/fstab
    ```

    chroot到新安装的系统

    ```text
    arch-chroot /mnt
    ```

    设置时区

    ```text
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    ```

    设置硬件时间

    ```text
    hwclock --systohc
    ```

    本地化
    安装vim和终端字体

    ```text
    pacman -S vim terminus-font
    ```

    编辑locale.gen

    ```text
    vim /etc/locale.gen
    ```

    删除如下两行开头的#

    ```text
    en_US.UTF-8 UTF-8
    
    zh_CN.UTF-8 UTF-8
    ```

    生成locale

    ```text
    locale-gen
    ```

    创建并编辑locale.conf

    ```text
    vim /etc/locale.conf
    ```

    按i进入编辑模式并输入如下语句后保存退出

    ```text
    LANG=en_US.UTF-8
    ```

11. **网络配置**
    设置hostname

    ```text
    vim /etc/hostname
    ----------------------------------
    我的主机名
    ```

    安装网络管理器

    ```text
    pacman -S networkmanager
    ```

    设置网络管理器开机自启

    ```text
    systemctl enable NetworkManager.service
    ```

12. 设置root密码

    ```text
    passwd
    New password:  # 请输入密码，这里不会有显示，这是正常现象
    Retype new password:
    passwd: password updated successfully
    ```

13. 引导加载程序
    根据cpu型号安装对应微码

    ```text
    pacman -S intel-ucode
    ```

    或者

    ```text
    pacman -S amd-ucode
    ```

    安装GRUB软件包

    ```text
    pacman -S grub efibootmgr
    
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
    ```

    生成GRUB配置

    ```text
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

    **一定要确保GRUB配置生成无误再重启！！！！**

14. 重启
输入exit退出chroot环境
输入reboot重启系统，此时拔出安装U盘

**至此Arch系统已经在电脑上安装完成了，虽然没有图形界面，但已经能够以命令行形式运行。**

---

### 系统配置

1. 登录root用户

    ```text
    Arch Linux 6.x.x-arch1-1 (tty1)
    xxxx login: root  # 请输入 root，以 root 身份登录
    Password: # 请输入密码，这里不会显示“*”，是正常现象
    [root@xxxx ~]# 
    ```

2. 连接互联网

    ```text
    [root@xxxx ~]# nmcli device wifi list  # 列出可连接的 WiFi
    [root@xxxx ~]# nmcli device wifi connect "SSID" password "密码"  # 连接 WiFi
    ```

3. 创建普通用户

    ```text
    [root@xxxx ~]# useradd -m -G wheel 用户名  # 创建用户，并为其创建家目录，将其加入 wheel 组
    [root@xxxx ~]# passwd 用户名
    New password:
    Retype new password:
    passwd: password updated successfully
    ```

4. 权限提升
    安装sudo

    ```text
    [root@xxxx ~]# pacman -S sudo
    ```

    编辑 /etc/sudoers

    ```text
    [root@xxxx ~]# vim /etc/sudoers
    ```

    去掉“%wheel ALL=(ALL:ALL) ALL”前面的“#”
    现在我们切换身份到新创建的用户。

    ```text
    [root@xxxx ~]# su - 用户名
    [用户名@xxxx ~]$ 
    ```

5. 配置pacman
    编辑/etc/pacman.conf文件
    首先，为了启用颜色显示和并行下载，请取消 Color 和 ParallelDownloads = 5 之前的“#”。
    其次，如果您需要在 Arch Linux 中运行 Steam 和 Wine（一款模拟器，可以在 Linux 中运行 Windows 程序），请启用 multilib 仓库。该仓库提供 32 位软件包。请搜索“multilib”，取消下述两行之前的“#”。
    然后，推荐您启用 archlinuxcn 仓库。该仓库由 Arch Linux CN 社区维护，提供了大量适合国人使用的软件包。请在文件最后新建一行，输入

    ```text
    /etc/pacman.conf
    ----------------
    [archlinuxcn]
    Server = https://mirrors.bfsu.edu.cn/archlinuxcn/$arch
    ```

    之后，保存并退出。请更新本地软件数据库并安装 archlinuxcn-keyring

    ```text
    pacman -Syyu archlinuxcn-keyring
    ```

6. 安装图形界面
    是的，终于进行到安装图形界面的步骤了，在这里不推荐使用Gnome，更不推荐配置更加复杂，适配更差的其他桌面环境，仅建议使用完整的KDE桌面，方便入门学习使用。
    安装Intel核显驱动，如果是AMD处理器请自行搜索

    ```text
    pacman -S xf86-video-intel
    ```

    安装Xorg，sddm并设置开机自启

    ```text
    pacman -S xorg sddm
    systemctl enable sddm
    ```

    安装KDE桌面及附属组件应用

    ```text
    pacman -S plasma kde-applications
    ```
    由于KDE桌面应用太多，我建议只安装桌面和几个必须的应用，其他应用等需要时再酌情安装。
    ```text
    pacman -S plasma-desktop konsole dolphin
    ```

    再次重启系统，就能通过sddm和kde图形界面进入系统了，注意，此时输入的密码已经是之前设置的普通用户的密码，而不是最开始设置的root用户密码。

7. 中文设置及中文输入法
    如上安装完成后，在kde设置中应该已经可以选择Chinese语言选项，如果没有，请参考[简体中文本地化)](https://wiki.archlinuxcn.org/wiki/%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87%E6%9C%AC%E5%9C%B0%E5%8C%96)中的配置说明中的locale部分进行配置，对于其他软件中的中文化配置，在此网页中也有详细说明。
    中文输入法目前推荐使用fcitx5，安装及配置参考：[Fcitx5](https://wiki.archlinuxcn.org/wiki/Fcitx5)
    日常使用中需要Microsoft相关字体的配置参考：[微软字体](https://wiki.archlinuxcn.org/wiki/%E5%BE%AE%E8%BD%AF%E5%AD%97%E4%BD%93)，推荐使用复制文件的方法，较快

8. aur源的使用及配置
    由于Linux本来可用软件数就不多，分发又复杂，因此有很大一部分软件由arch用户自行编译发布，即aur仓库。通常使用yay来下载aur中的软件。
    安装yay

    ```text
    sudo pacman -S yay
    ```

    之后使用yay安装，卸载，搜索应用的方法与pacman大致相同，可自行搜索相关页面，参考[pacman - Arch Linux 中文维基](https://wiki.archlinuxcn.org/wiki/Pacman)


**建议：定期（至少每周，推荐每天）运行系统更新命令进行滚动更新。**

```text
    yay -Syu
```

## Archlinux 安装后配置

### 常用应用

### KDE配置

待续