title: sd与mmc刷写指南
---
## Env

Host: Ubuntu 22.04

GParted 1.3.1

## SD卡

首先插入sd卡，打开GParted工具，没有可以下载一个：

```bash
❯ sudo apt install gpart
```

选择sd卡设备，我这里是sdb，因机器而异。

选择菜单栏的`设备`->`创建分区表`，选择msdos分区表。

[![2024-09-20-17-52-19.png](https://i.postimg.cc/rFV6F05D/2024-09-20-17-52-19.png)](https://i.postimg.cc/rFV6F05D/2024-09-20-17-52-19.png)

创建完成之后，新建两个分区。

一个fat16的，用于放内核镜像和dtb文件以及uboot保存的env文件。

[![2024-09-20-17-55-43.png](https://i.postimg.cc/wMFGs2D7/2024-09-20-17-55-43.png)](https://postimg.cc/bSDT7QCh)

一个ext4用来存放根文件系统。

[![2024-09-20-17-55-55.png](https://i.postimg.cc/h4MysC7G/2024-09-20-17-55-55.png)](https://postimg.cc/w3yQTQ2K)

完成之后点击绿色小对勾，让更改生效。

[![2024-09-20-17-56-02.png](https://i.postimg.cc/hPjC5991/2024-09-20-17-56-02.png)](https://postimg.cc/Dm9dW4pS)

之后便可以在`/dev`下看到对应的分区：

```bash
❯ ls /dev/sdb*
/dev/sdb  /dev/sdb1  /dev/sdb2
```

下载uboot镜像：

```bash
❯ sudo dd if=u-boot-sunxi-with-spl.bin of=/dev/sdb bs=1024 seek=8
```

存放内核镜像和设备树：

```
❯ sudo mount /dev/sdb1 /mnt
❯ sudo cp arch/arm/boot/zImage /mnt
❯ sudo cp arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dtb /mnt
❯ sudo umount /mnt
```

存放根文件系统：

```bash
❯ sudo mount /dev/sdb2 /mnt
❯ sudo tar -xvzf ubuntu22.04.tar.gz -C /mnt
❯ sudo umount /mnt
```

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

