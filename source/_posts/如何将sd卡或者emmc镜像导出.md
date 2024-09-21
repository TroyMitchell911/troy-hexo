title: 如何将sd卡或者emmc镜像导出
date: '2024-09-21 12:30:52'
updated: '2024-09-21 12:30:54'
tags:
  - ubuntu
  - kernel
  - linux
categories:
  - kernel
---
## SD卡

插入`sd卡`后，会在`/dev`下看到`sdX`的文件，我这里是`sdb`：

```bash
❯ ls /dev/sdb*
/dev/sdb  /dev/sdb1  /dev/sdb2
```

从这个信息可以知道镜像至少分区为了`boot`和`rootfs分区`.

使用dd命令导出：

```bash
❯ sudo dd if=/dev/sdb of=sd.img bs=4M status=progress
[sudo] troy 的密码： 
63753420800字节（64 GB，59 GiB）已复制，652 s，97.8 MB/s 
记录了15218+1 的读入
记录了15218+1 的写出
63831015424字节（64 GB，59 GiB）已复制，653.304 s，97.7 MB/s
```

导出后需要[缩减分区](##缩减分区)

## EMMC

对于EMMC操作复杂一些，需要修改uboot，打开CONFIG_CMD_USB_MASS_STORAGE的配置项。

进入fel模式将uboot下载进入内存并运行：

```bash
❯ sudo sunxi-fel uboot ./u-boot-sunxi-with-spl.bin
```

在uboot自动启动内核前打断，并且输入如下命令：

```bash
=> ums 2 mmc 1
```

这个命令是将mmc的1号设备通过usb 2号挂载出去。就相当于电脑插了个U盘。

我这里mmc的1号设备是emmc，usb2号是otg接口，需要根据实际情况修改。

使用如下命令可查询对应的接口号：

```bash
=> mmc list
=> usb tree
# 或者是usb info
```

然后电脑就能看到了`/dev/sdX`，随后的操作与SD卡相同。

## 缩减分区

由于镜像中有多个分区，所以先关联到回环设备：

```bash
❯ sudo losetup -Pf sd.img
❯ ls /dev/loop*p*
/dev/loop15p1  /dev/loop15p2
```

这里就挂出来了两个分区，`p1`是`boot`，`p2`是`rootfs`，我们仅需要将rootfs分区缩减，因为他是大头。

打开`Ubuntu`自带的磁盘工具，调整`rootfs`大小为最小:

[![2024-09-21-11-30-46.png](https://i.postimg.cc/yYRYvWTN/2024-09-21-11-30-46.png)](https://postimg.cc/mtbsrLJx)

调整完成后如下图所示：

[![2024-09-21-11-32-23.png](https://i.postimg.cc/ZR0J3qR4/2024-09-21-11-32-23.png)](https://postimg.cc/XGSM64dz)

已经成功调整了分区 `/dev/loop15p2` 的大小到 1.7GB，但 `sd.img` 仍然保持 60GB，原因是文件系统和分区的大小确实已经改变，但镜像文件本身的大小并没有自动缩减：

```bash
❯ sudo fdisk -l /dev/loop15
Disk /dev/loop15：59.45 GiB，63831015424 字节，124669952 个扇区
单元：扇区 / 1 * 512 = 512 字节
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x8c4a4ad7

设备          启动  起点    末尾    扇区  大小 Id 类型
/dev/loop15p1       2048   67583   65536   32M  6 FAT16
/dev/loop15p2      67584 3588095 3520512  1.7G 83 Linux
❯ ls -lh sd.img
-rw-r--r-- 1 root root 60G  9月 21 11:31 sd.img
```

需要手动缩减镜像文件的大小，使其匹配实际使用的分区大小。

 `/dev/loop15p2` 的结束位置是扇区 `3588095`，镜像文件的大小可以通过这个结束位置来确定。

由于每个扇区大小为 512 字节，可以计算出新的镜像文件大小：

```bash
新镜像大小 = (最后一个扇区号 + 1) * 扇区大小
```

最后一个扇区是 `3588095`，所以新的镜像大小为：

```bash
新镜像大小 = (3588095 + 1) * 512 = 1838080512 字节
```

使用`truncate`缩减镜像：

```bash
❯ sudo truncate --size=$(( (3588095 + 1) * 512 )) sd.img
❯ ls -lh sd.img
-rw-r--r-- 1 root root 1.8G  9月 21 11:35 sd.img
```

完成后解除回环设备关联：

```bash
❯ sudo losetup -d /dev/loop15
```

## 验证

将刚才制作好的镜像烧写进入，这里以sd卡为例：

```bash
❯ sudo dd if=sd.img of=/dev/sdb status=progress
1827463680字节（1.8 GB，1.7 GiB）已复制，230 s，7.9 MB/s 
记录了3588096+0 的读入
记录了3588096+0 的写出
1837105152字节（1.8 GB，1.7 GiB）已复制，230.959 s，8.0 MB/s
```

启动系统查看是否正确。

```bash
root@a33-vstar:~# lsblk
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
mmcblk0      179:0    0 59.4G  0 disk 
├─mmcblk0p1  179:1    0   32M  0 part 
└─mmcblk0p2  179:2    0  1.7G  0 part /
mmcblk1      179:8    0  3.7G  0 disk 
├─mmcblk1p1  179:9    0   32M  0 part 
└─mmcblk1p2  179:10   0  3.7G  0 part 
mmcblk1boot0 179:16   0    2M  1 disk 
mmcblk1boot1 179:24   0    2M  1 disk 
root@a33-vstar:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       1.4G  568M  741M  44% /
tmpfs           232M     0  232M   0% /dev/shm
tmpfs            93M  2.8M   90M   3% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs            47M     0   47M   0% /run/user/0
```

可以看到mmcblk0p2确实变成了1.7G，但是剩余的空间并没有使用到，此时就需要扩展文件系统分区：

```bash
root@a33-vstar:~# sudo parted /dev/mmcblk0 --script resizepart 2 100%
Kernel not configured for semaphores (System V IPC). Not using udev synchronisation code.
root@a33-vstar:~# sudo resize2fs /dev/mmcblk0p2
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mmcblk0p2 is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 4
The filesystem on /dev/mmcblk0p2 is now 15575296 (4k) blocks long.
root@a33-vstar:~# lsblk
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
mmcblk0      179:0    0 59.4G  0 disk 
├─mmcblk0p1  179:1    0   32M  0 part 
└─mmcblk0p2  179:2    0 59.4G  0 part /
mmcblk1      179:8    0  3.7G  0 disk 
├─mmcblk1p1  179:9    0   32M  0 part 
└─mmcblk1p2  179:10   0  3.7G  0 part 
mmcblk1boot0 179:16   0    2M  1 disk 
mmcblk1boot1 179:24   0    2M  1 disk
```

