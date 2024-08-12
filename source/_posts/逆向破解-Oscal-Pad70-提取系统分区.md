title: '[逆向破解]Oscal Pad70---提取系统分区'
date: '2024-08-12 20:23:25'
updated: '2024-08-12 21:23:46'
tags:
  - kernel
  - linux
  - hack
  - rockchip
  - rk3566
categories:
  - kernel
  - reverse
---
## Starting

Oscal Pad70实物图：

![oscal pad70](https://i.imghippo.com/files/Ct0oQ1723469150.jpg)

由官网可知，Oscal Pad70是使用rk3566作为主控：

![oscal pad70 website](https://i.imghippo.com/files/wqrko1723469276.png)

这台设备并没有关闭adb，使用adb命令可以很轻松的查看到adb的设备。

```bash
❯ adb devices
List of devices attached
Pad70PROL02309200121	device

❯ adb shell
Pad70_Pro:/ $ cat /proc/version                                                                              
Linux version 5.10.168-android13-4-00008-g33b1e2eb04dc-ab10159773 (build-user@build-host) (Android (8508608, based on r450784e) clang version 14.0.7 (https://android.googlesource.com/toolchain/llvm-project 4c603efb0cca074e9238af8b4106c30add4418f6), LLD 14.0.7) #1 SMP PREEMPT Thu May 11 18:17:05 UTC 2023
```

不出意料的是5.10版本的内核。

## 备份分区

为了防止在后续移植过程中出问题，先备份一下分区。

使用如下命令查看分区：

```bash
Pad70_Pro:/ $ ls /dev/block/by-name -l
total 0
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 backup -> /dev/block/mmcblk2p19
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 baseparameter -> /dev/block/mmcblk2p23
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 boot_a -> /dev/block/mmcblk2p17
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 boot_b -> /dev/block/mmcblk2p18
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 cache -> /dev/block/mmcblk2p20
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 dtbo_a -> /dev/block/mmcblk2p13
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 dtbo_b -> /dev/block/mmcblk2p14
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 frp -> /dev/block/mmcblk2p22
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 init_boot_a -> /dev/block/mmcblk2p11
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 init_boot_b -> /dev/block/mmcblk2p12
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 metadata -> /dev/block/mmcblk2p21
lrwxrwxrwx 1 root root 20 2024-08-11 20:21 misc -> /dev/block/mmcblk2p6
lrwxrwxrwx 1 root root 18 2024-08-11 20:21 mmcblk2 -> /dev/block/mmcblk2
lrwxrwxrwx 1 root root 23 2024-08-11 20:21 mmcblk2boot0 -> /dev/block/mmcblk2boot0
lrwxrwxrwx 1 root root 23 2024-08-11 20:21 mmcblk2boot1 -> /dev/block/mmcblk2boot1
lrwxrwxrwx 1 root root 20 2024-08-11 20:21 resource_a -> /dev/block/mmcblk2p7
lrwxrwxrwx 1 root root 20 2024-08-11 20:21 resource_b -> /dev/block/mmcblk2p8
lrwxrwxrwx 1 root root 20 2024-08-11 20:21 security -> /dev/block/mmcblk2p1
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 super -> /dev/block/mmcblk2p24
lrwxrwxrwx 1 root root 20 2024-08-11 20:21 trust_a -> /dev/block/mmcblk2p4
lrwxrwxrwx 1 root root 20 2024-08-11 20:21 trust_b -> /dev/block/mmcblk2p5
lrwxrwxrwx 1 root root 20 2024-08-11 20:21 uboot_a -> /dev/block/mmcblk2p2
lrwxrwxrwx 1 root root 20 2024-08-11 20:21 uboot_b -> /dev/block/mmcblk2p3
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 userdata -> /dev/block/mmcblk2p25
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 vbmeta_a -> /dev/block/mmcblk2p15
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 vbmeta_b -> /dev/block/mmcblk2p16
lrwxrwxrwx 1 root root 20 2024-08-11 20:21 vendor_boot_a -> /dev/block/mmcblk2p9
lrwxrwxrwx 1 root root 21 2024-08-11 20:21 vendor_boot_b -> /dev/block/mmcblk2p10
```

可以看到文件超级多，没办法，是个体力活了。

从系统分区的_a和_b能够看出来，使用的是采用了 A/B（双系统）分区结构。这种分区结构常用于安卓设备，以实现无缝系统更新。

> 双分区： 在 A/B 分区结构中，关键的系统分区（如 boot、system、vendor 等）被复制成两份，分别为 A（_a 后缀）和 B（_b 后缀）分区。
系统更新： 当有新的系统更新时，系统会将更新写入未使用的分区（例如当前使用 A 分区时会更新 B 分区）。更新完成后，设备会切换到更新后的分区启动。

使用以下命令查看当前正在使用的分区：

```bash
Pad70_Pro:/mnt/backup # getprop ro.boot.slot_suffix
_a
```

正在使用的分区是_a，所以_b的我们就不用提取了。

使用如下命令提取分区：

```bash
Pad70_Pro:/ # dd if=/dev/block/mmcblk2p19 of=backup.img                                                        
dd: backup.img: Read-only file system
0+0 records in
0+0 records out
0 bytes (0 B) copied, 0.000704 s, 0 B/s
```

发现根目录是以只读方式挂载的，所以两种方案：
- 重新挂载
- 找一个可写的目录

这里偷懒使用/data，不重新挂载了：

```bash
cd /data && mkdir back && cd back

dd if=/dev/block/mmcblk2p19 of=backup.img   
dd if=/dev/block/mmcblk2p23 of=baseparameter    
dd if=/dev/block/mmcblk2p17 of=boot_a.img      
dd if=/dev/block/mmcblk2p20 of=cache.img    
dd if=/dev/block/mmcblk2p13 of=dtbo_a.img       
dd if=/dev/block/mmcblk2p22 of=frp.img
dd if=/dev/block/mmcblk2p11 of=init_boot_a.img
dd if=/dev/block/mmcblk2p21 of=metadata.img
dd if=/dev/block/mmcblk2p6 of=misc.img
dd if=/dev/block/mmcblk2boot0 of=mmcblk2boot0.img
dd if=/dev/block/mmcblk2boot1 of=mmcblk2boot1.img
dd if=/dev/block/mmcblk2p7 of=resource_a.img
dd if=/dev/block/mmcblk2p1 of=security.img
dd if=/dev/block/mmcblk2p24 of=super.img
dd if=/dev/block/mmcblk2p4 of=trust_a.img
dd if=/dev/block/mmcblk2p2 of=uboot_a.img
dd if=/dev/block/mmcblk2p15 of=vbmeta_a.img
dd if=/dev/block/mmcblk2p9 of=vendor_boot_a.img
dd if=/dev/block/mmcblk2p25 of=userdata.img && dd if=/dev/block/mmcblk2 of=mmcblk2.img
```

使用pull命令拉取到本地文件夹：

```bash
❯ adb pull /data/back/
```

## Ref

[1]: https://www.oscal.hk/pad70