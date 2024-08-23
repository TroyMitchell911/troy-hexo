title: OpenHarmony on rk3568使能can
date: '2024-08-22 13:41:29'
updated: '2024-08-22 13:41:31'
tags:
  - openHarmony
  - kernel
  - linux
  - rk3568
categories:
  - kernel
  - rockchip
---
## Env

OH: v3.2.3
chip: rk3568

## Content

首先查看源码中是否具有`CAN_ROCKCHIP`选项：

```bash
❯ find -name "Kconfig" -exec grep -n "CAN_ROCKCHIP" {} +
❯ 
```

发现什么都没有..好吧，那看到官方的rk3568的补丁并没有打到这里，需要自己进行适配。

去rockchip的kernel仓库找到关于can的部分：https://github.com/rockchip-linux/kernel/blob/develop-5.10/drivers/net/can/rockchip/

进入到我们的内核工作目录，没有Makefile创建的可以见这篇文章：https://blog.505218.xyz/2024/08/16/rk3568%E7%A7%BB%E6%A4%8DopenHarmony-v3-2-3-%E7%B3%BB%E7%BB%9F%E7%A7%BB%E6%A4%8D/

将刚才仓库的文件无论用什么方式放到`driver/net/can/rockchip`目录下，随后进行`git`操作:

```bash
❯ git add drivers/net/can/Makefile
❯ git add drivers/net/can/Kconfig
❯ git commit -m "can init"
```

并且修改`driver/net/can/Makefile`和`driver/net/can/Kconfig`:

```bash
❯ diff drivers/net/can/Makefile mmm
32d31
< obj-$(CONFIG_CAN_ROCKCHIP)	+= rockchip/
❯ diff drivers/net/can/Kconfig kkk
181d180
< source "drivers/net/can/rockchip/Kconfig"
```

随后修改`config`文件：

```bash
❯ diff arch/arm64/configs/rockchip_linux_defconfig ccc
6858,6860d6857
< CONFIG_CAN=y
< CONFIG_CAN_ROFKCHIP=y
< CONFIG_CANFD_ROCKCHIP=y
```

继续`git`操作:

```bash
❯ git add drivers/net/can/Makefile
❯ git add drivers/net/can/Kconfig
❯ git add drivers/net/can/rockchip
❯ git commit -m "add can support"
```

生成`patch`：

```bash
❯ git format-patch --subject-prefix='PATCH' -n -s -1
0001-add-can-support.patch
❯ cp 0001-add-can-support.patch /mnt/v3.2.3/kernel/linux/patches/linux-5.10/rk3568_patch/add-can-support.patch && rm *.patch
```

修改编译脚本：

```bash
❯ diff device/board/hihope/rk3568/kernel/build_kernel.sh device/board/hihope/rk3568/kernel/build_kernel.sh.1
34,35c34
< 		${ROOT_DIR}/kernel/linux/patches/linux-5.10/rk3568_patch/add-jl201-drivers.patch
< 		${ROOT_DIR}/kernel/linux/patches/linux-5.10/rk3568_patch/add-can-support.patch"
---
> 		${ROOT_DIR}/kernel/linux/patches/linux-5.10/rk3568_patch/add-jl201-drivers.patch"
```

修改设备树：

```bash

❯ vim device/board/hihope/rk3568/kernel/dts/rk3568-xxx.dtsi

# 以下内容添加
&can2 {
	compatible = "rockchip,rk3568-can-2.0";
	pinctrl-names = "default";
	pinctrl-0 = <&can2m0_pins>;
	status = "okay";
};
```

## Ref

https://blog.505218.xyz/2024/08/16/rk3568%E7%A7%BB%E6%A4%8DopenHarmony-v3-2-3-%E7%B3%BB%E7%BB%9F%E7%A7%BB%E6%A4%8D/
https://github.com/rockchip-linux/kernel/blob/develop-5.10/drivers/net/can/rockchip/