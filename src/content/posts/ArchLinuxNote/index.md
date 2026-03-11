---
title: "Linux常用命令 - 小记"
published: 2026-03-08
description: ""
image: "cover.png"
tags: [记录]
category: "分享"
draft: false
lang: "zh-CN"
---

# 网络

## ip address 命令

查看本机网络接口

```sh + {1,2} with wrap=false collapse={3-11,20-25} mark="ip a" mark="10.218.0.141/15"
┬─[bluore@ArchLinux-bluore:~]─[04:24:18 PM]
╰─>$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp4s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500qdisc fq_codel state DOWN group default qlen 1000
    link/ether 1c:83:41:ce:04:ea brd ff:ff:ff:ff:ff:ff
    altname enx1c8341ce04ea
3: wlo1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 98:bd:80:34:02:a6 brd ff:ff:ff:ff:ff:ff
    altname wlp3s0
    altname wlx98bd803402a6
    inet 10.218.0.141/15 brd 10.219.255.255 scope global dynamic noprefixroute wlo1
       valid_lft 603914sec preferred_lft 603914sec
    inet6 fe80::6ff3:52be:8bcf:c275/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
4: singbox_tun: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 9000 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none
    inet 172.18.0.1/30 brd 172.18.0.3 scope global singbox_tun
       valid_lft forever preferred_lft forever
    inet6 fe80::3dd2:db03:6ace:55c5/64 scope link stable-privacy proto kernel_ll
       valid_lft forever preferred_lft forever`
```

上面是开启了`TUN（虚拟网卡）`代理的运行情况，如果需要卸载`虚拟网卡`执行`ip link delete <虚拟网卡名>`

## iwd 连接网络

```sh title="进入控制界面" 
iwctl
```

```sh title="列出设备列表" 
device list
```

```sh title="扫描网络" mark="wlan0"
# 填写你扫描到的设备
station wlan0 scan
station wlan0 get-networks
```

```sh title="连接网络" 
station wlan0 connect <网络名称>
# 回车后输入密码
```

示例：

```txt {1-2,6,13,14,24,27} with wrap=false collapse={2-27}
┬─[bluore@ArchLinux-bluore:~]─[05:17:44 PM]
╰─>$ iwctl
NetworkConfigurationEnabled: disabled
StateDirectory: /var/lib/iwd
Version: 3.11
[iwd]# device list
                                    Devices                                   *
--------------------------------------------------------------------------------
  Name                  Address               Powered     Adapter     Mode
--------------------------------------------------------------------------------
  wlan0                 98:bd:80:34:02:a6     on          phy0        station

[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
                               Available networks                             *
--------------------------------------------------------------------------------
      Network name                      Security            Signal
--------------------------------------------------------------------------------
      NSU-SDN                           open                ****
      此消彼长                          psk                 ****
      Bluore's phone                    psk                 ****
      来蹭你爹的网                      psk                 ****

[iwd]# station wlan0 connect "Bluore's phone"
Type the network passphrase for Bluore's phone psk.
Passphrase: ********
[iwd]# exit
```

# 分区

## lsblk

```sh 
lsblk -f
```

```txt {1-2} with wrap=false collapse={2-22}
┬─[bluore@ArchLinux-bluore:~]─[06:15:02 PM]
╰─>$ lsblk -f
NAME FSTYPE FSVER LABEL    UUID      FSAVAIL FSUSE% MOUNTPOINTS
zram0
     swap   1     zram0    9fb3a373-be98-4e93-aba6-88fdb3a6f25c                [SWAP]
nvme0n1
│
├─nvme0n1p1
│    vfat   FAT32          7049-B703       140.4M    30% /efi
├─nvme0n1p2
│
├─nvme0n1p3
│    ntfs         System   249A76BE9A768C54
├─nvme0n1p4
│    ntfs                  E29473BC94739231
├─nvme0n1p5
│    ntfs         Software B65659F95659BB37
├─nvme0n1p6
│    btrfs                 032c17e9-7417-4345-b04f-5ee0561a7ced   30.6G    83% /home
│                     /
└─nvme0n1p7
     btrfs                 9fea03d7-4c78-47cc-83ea-940ff1afc944
```

## fdisk & cfdisk

```sh 
fdisk -l
```

```txt {1-2} with wrap=false collapse={2-26}
┬─[bluore@ArchLinux-bluore:~]─[06:16:43 PM]
╰─>$ sudo fdisk -l
Disk /dev/nvme0n1: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: KINGSTON SNV2S1000G
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: C1F0B2B5-3ACF-4502-A82D-5DB9C5B64466

Device              Start        End   Sectors   Size Type
/dev/nvme0n1p1         40     409639    409600   200M EFI System
/dev/nvme0n1p2     409640     442407     32768    16M Microsoft reserved
/dev/nvme0n1p3     444416  313382911 312938496 149.2G Microsoft basic data
/dev/nvme0n1p4  313382912  315015167   1632256   797M Windows recovery environment
/dev/nvme0n1p5  315015168 1124514447 809499280   386G Microsoft basic data
/dev/nvme0n1p6 1544091648 1953523711 409432064 195.2G Linux filesystem
/dev/nvme0n1p7 1124515840 1544091647 419575808 200.1G Linux filesystem

Partition table entries are not in disk order.


Disk /dev/zram0: 30.52 GiB, 32769048576 bytes, 8000256 sectors
Units: sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```

## GRUB 更新

```sh 
grub-install --target=x86_64-efi --efi-directory=<EFI挂载目录> --bootloader-id=<启动项名称>
grub-mkconfig -o /boot/grub/grub.cfg
```

> 可用于在 live 环境中修复EFI启动项


# 杂项

## 配置镜像源 **pacman**

```sh title="测速并更新pacman的镜像源" {"将12小时内更新过的10个中国镜像替换到pacman":1-2}
#
reflector -a 12 -c cn -f 10 --sort score --v --save /etc/pacman.d/mirrorlist
```

## 获取系统相关信息

```sh 
uanme -a
```

```sh 
fastfetch
```

