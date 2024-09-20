title: '[全志A33-Vstar]Kernel'
date: '2024-09-20 22:49:02'
updated: '2024-09-20 22:49:04'
tags:
  - a33
  - kernel
  - allwinner
---
**该文章基于该uboot启动**：[传送门](https://blog.505218.xyz/2024/09/13/%E5%85%A8%E5%BF%97A33-Uboot/)

## Env

cpu: allwinner a33
board: vstar
host: ubuntu 22.04

需要安装交叉编译工具链：

```bash
❯ sudo apt install gcc-arm-none-eabi
```

## FEL模式

通过FEL模式可以启动uboot或将内核镜像等文件下载到内存，是个很方便的功能。

若要使用fel系列的工具，需要先安装：

```bash
❯ sudo apt-get install sunxi-tools
```

vstar开发板进入fel的方式有两种：

- 按住power键不松手，随后按reset，等待1s后放开power键
- 按住vol + 键不松手，随后按reset，连续短按5-10次power键后有一个灯闪烁一下，此时松开vol+键即可进入。

fel烧写uboot命令：

```bash
❯ sudo sunxi-fel uboot ./u-boot-sunxi-with-spl.bin
```

## mainline

首先下载主线kernel的源码：

```bash
❯ git clone git@github.com:torvalds/linux.git
```

配置文件的话，在`arch/arm/configs`目录下只有一个`sunxi_defconfig`是我们能够使用的，那么就使用这个配置文件作为基准。

设备树就选择sinlinx的，与vstar开发板相近。

与uboot套路类似，先拷贝一份配置文件和设备树作为vstar开发板的：

```bash
❯ cp arch/arm/configs/sunxi_defconfig arch/arm/configs/a33_vstar_defconfig
❯ cp arch/arm/boot/dts/allwinner/sun8i-a33-sinlinx-sina33.dts arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts
```

修改Makefile支持该设备树：

```bash
diff --git a/arch/arm/boot/dts/allwinner/Makefile b/arch/arm/boot/dts/allwinner/Makefile
index cd0d044882cf..d548f4a2621a 100644
--- a/arch/arm/boot/dts/allwinner/Makefile
+++ b/arch/arm/boot/dts/allwinner/Makefile
@@ -215,6 +215,7 @@ dtb-$(CONFIG_MACH_SUN8I) += \
        sun8i-a33-olinuxino.dtb \
        sun8i-a33-q8-tablet.dtb \
        sun8i-a33-sinlinx-sina33.dtb \
+       sun8i-a33-vstar.dtb \
        sun8i-a83t-allwinner-h8homlet-v2.dtb \
        sun8i-a83t-bananapi-m3.dtb \
        sun8i-a83t-cubietruck-plus.dtb \
```

使用刚才拷贝的配置文件进行编译：

```bash
```