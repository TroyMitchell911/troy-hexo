title: rk3568 uboot distro_cmd
date: '2024-08-23 16:12:48'
updated: '2024-08-26 10:34:18'
tags:
  - rk3568
  - rockchip
  - linux
  - kernel
categories:
  - kernel
  - rockchip
---
## bootcmd

在`uboot`的`shell`中使用`printenv`命令可以看到`bootcmd`的字符串：

```bash
=> printenv bootcmd
bootcmd=boot_android ${devtype} ${devnum};boot_fit;bootrkp;run distro_bootcmd;
```

命令解析：
    - `boot_android`: 查找并尝试启动安卓镜像。
    - `boot_fix`: 查找并尝试启动fit格式镜像。
    - `bootrkp`: 查找并尝试启动rk分区镜像。
    - `run distro_bootcmd`: 这就是这篇文章的主角了，distro_bootcmd可以根据不同介质去启动`kernel`内核，比如`kernel`可以在sd卡，emmc上，都可以顺利启动。

以上提到的`bootcmd`可以在`include/configs/rockchip-common.h`中看到相关定义：

```c
#if defined(CONFIG_AVB_VBMETA_PUBLIC_KEY_VALIDATE)
#define RKIMG_BOOTCOMMAND			\
	"boot_android ${devtype} ${devnum};"
#elif defined(CONFIG_FIT_SIGNATURE)
#define RKIMG_BOOTCOMMAND			\
	"boot_fit;"
#else
#define RKIMG_BOOTCOMMAND			\
	"boot_android ${devtype} ${devnum};"	\
	"boot_fit;"				\
	"bootrkp;"				\
	"run distro_bootcmd;"
#endif
```

## distro_bootcmd

现在来重点分析一下`distro_bootcmd`，首先使用打印出`distro_bootcmd`:

```bash
=> printenv distro_bootcmd 
distro_bootcmd=for target in ${boot_targets}; do run bootcmd_${target}; done

```

在`uboot`源码中找到对应实现，可以看到`BOOTENV_SET_SCSI_NEED_INIT`这个宏定义是空的，所以不必深究：

```c
#define BOOTENV \
    BOOTENV_BOOT_TARGETS
    ...
	BOOT_TARGET_DEVICES(BOOTENV_DEV)                                  \
	\
	"distro_bootcmd=" BOOTENV_SET_SCSI_NEED_INIT                      \
		"for target in ${boot_targets}; do "                      \
			"run bootcmd_${target}; "                         \
		"done\0"

```

### BOOTENV

在这个`BOOTENV`宏定义中，定义了`distro_bootcmd`的值，`BOOTENV`在`include/configs/rk3568_common.h`中被使用到：

```c#define CONFIG_EXTRA_ENV_SETTINGS \
	ENV_MEM_LAYOUT_SETTINGS \
	"partitions=" PARTS_RKIMG \
	ROCKCHIP_DEVICE_SETTINGS \
	RKIMG_DET_BOOTDEV \
	BOOTENV
#endif
```

而`CONFIG_EXTRA_ENV_SETTINGS`又在`include/env_default.h`中被使用到：

```c
#ifdef DEFAULT_ENV_INSTANCE_EMBEDDED
env_t environment __UBOOT_ENV_SECTION__ = {
	ENV_CRC,	/* CRC Sum */
#ifdef CONFIG_SYS_REDUNDAND_ENVIRONMENT
	1,		/* Flags: valid */
#endif
	{
#elif defined(DEFAULT_ENV_INSTANCE_STATIC)
static char default_environment[] = {
#else
const uchar default_environment[] = {
#endif
#ifdef	CONFIG_USE_BOOTARGS
	"bootargs="	CONFIG_BOOTARGS			"\0"
#endif
#ifdef	CONFIG_BOOTCOMMAND
	"bootcmd="	CONFIG_BOOTCOMMAND		"\0"
#endif
#endif
...
#ifdef	CONFIG_EXTRA_ENV_SETTINGS
	CONFIG_EXTRA_ENV_SETTINGS
#endif
	"\0"
#ifdef DEFAULT_ENV_INSTANCE_EMBEDDED
	}
#endif
```

### TODO 了解这个数组

回到`distro_bootcmd`：

```c
#define BOOTENV \
    BOOTENV_BOOT_TARGETS
    ...
	BOOT_TARGET_DEVICES(BOOTENV_DEV)                                  \
	\
	"distro_bootcmd=" BOOTENV_SET_SCSI_NEED_INIT                      \
		"for target in ${boot_targets}; do "                      \
			"run bootcmd_${target}; "                         \
		"done\0"

```

这里其实就是一个循环，循环遍历` ${boot_targets}`并执行`bootcmd_${target}`,查看` ${boot_targets}`相关定义：

```c
// path: include/config_distro_bootcmd.h
#define BOOTENV_DEV_NAME(devtypeu, devtypel, instance) \
	BOOTENV_DEV_NAME_##devtypeu(devtypeu, devtypel, instance)
#define BOOTENV_BOOT_TARGETS \
	"boot_targets=" BOOT_TARGET_DEVICES(BOOTENV_DEV_NAME) "\0"
```

### BOOTENV_BOOT_TARGETS生成boot_targets字符串

通过`BOOTENV_BOOT_TARGETS`宏定义生成了`boot_targets`字符串：

```c
// path: include/config_distro_bootcmd.h
#define BOOTENV_DEV_NAME(devtypeu, devtypel, instance) \
	BOOTENV_DEV_NAME_##devtypeu(devtypeu, devtypel, instance)
#define BOOTENV_BOOT_TARGETS \
	"boot_targets=" BOOT_TARGET_DEVICES(BOOTENV_DEV_NAME) "\0"
```

> [!NOTE] 
> BOOTENV_BOOT_TARGETS也是在BOOTENV中定义的

`BOOTENV_BOOT_TARGETS`调用了`BOOT_TARGET_DEVICES`宏定义，`BOOT_TARGET_DEVICES`就是在对应的芯片上或芯片系列的定义了，不同的芯片应有不同的实现：

```c
// path: include/configs/rockchip-common.h
#define BOOT_TARGET_DEVICES(func) \
	BOOT_TARGET_MMC(func) \
	BOOT_TARGET_MTD(func) \
	BOOT_TARGET_RKNAND(func) \
	BOOT_TARGET_USB(func) \
	BOOT_TARGET_PXE(func) \
	BOOT_TARGET_DHCP(func)
```

这里宏定义定义成了6个宏，这里只研究`BOOT_TARGET_MMC`，其他都是差不多的：

```c
// path: include/configs/rockchip-common.h
/* First try to boot from SD (index 1), then eMMC (index 0) */
#if CONFIG_IS_ENABLED(CMD_MMC)
	#define BOOT_TARGET_MMC(func) \
		func(MMC, mmc, 1) \
		func(MMC, mmc, 0)
#else
	#define BOOT_TARGET_MMC(func)
#endif
```

逐级展开：

```c
#define BOOTENV_DEV_NAME(devtypeu, devtypel, instance) \
	BOOTENV_DEV_NAME_##devtypeu(devtypeu, devtypel, instance)
#define BOOTENV_BOOT_TARGETS \
	"boot_targets=" BOOT_TARGET_DEVICES(BOOTENV_DEV_NAME) "\0"

    ->（这里就只展开mmc了，道理都一样）
    "boot_targets="  BOOT_TARGET_MMC(BOOTENV_DEV_NAME) "\0"
    ->
    "boot_targets="  BOOTENV_DEV_NAME(MMC, mmc, 1) \
		                        BOOTENV_DEV_NAME(MMC, mmc, 0)"\0"
    ->
    "boot_targets="  BOOTENV_DEV_NAME_MMC(MMC, mmc, 1) \
		                        BOOTENV_DEV_NAME_MMC(MMC, mmc, 0)"\0"

```

`BOOTENV_DEV_NAME_MMC`宏定义:

```c
// path: include/config_distro_bootcmd.h
#define BOOTENV_DEV_NAME_BLKDEV(devtypeu, devtypel, instance) \
	#devtypel #instance " "
#define BOOTENV_DEV_NAME_MMC	BOOTENV_DEV_NAME_BLKDEV
```

继续展开：

```c
#define BOOTENV_DEV_NAME(devtypeu, devtypel, instance) \
	BOOTENV_DEV_NAME_##devtypeu(devtypeu, devtypel, instance)
#define BOOTENV_BOOT_TARGETS \
	"boot_targets=" BOOT_TARGET_DEVICES(BOOTENV_DEV_NAME) "\0"

    ->（这里就只展开mmc了，道理都一样）
    "boot_targets="  BOOT_TARGET_MMC(BOOTENV_DEV_NAME) "\0"
    ->
    "boot_targets="  BOOTENV_DEV_NAME(MMC, mmc, 1) \
		                        BOOTENV_DEV_NAME(MMC, mmc, 0)"\0"
    ->
    "boot_targets="  BOOTENV_DEV_NAME_MMC(MMC, mmc, 1) \
		                        BOOTENV_DEV_NAME_MMC(MMC, mmc, 0)"\0"
    ->
    "boot_targets="  BOOTENV_DEV_NAME_BLKDEV(MMC, mmc, 1) \
		                        BOOTENV_DEV_NAME_BLKDEV(MMC, mmc, 0)"\0"
        ->
    "boot_targets="  mmc1 mmc0  "\0"
```

最后`boot_targets`就变成了下面这样，同时在`uboot`的`shell`中验证也是这样：

```bash
=> printenv boot_targets 
boot_targets=mmc1 mmc0 mtd2 mtd1 mtd0 usb0 pxe dhcp 
```

`boot_targets`分析完了，但这只是我们`distro_bootcmd`循环的变量而已，还有一个`run bootcmd_${target};`没有分析，通过`boot_targets`能知道，这运行的大概就是`bootcmd_mmc0`, `bootcmd_mmc1`这样的东西。

### 生成distro_cmd运行命令

在`BOOTENV`中有一个宏调用是`BOOT_TARGET_DEVICES(BOOTENV_DEV)`（上文BOOTENV代码中贴了），现在回顾一下`BOOT_TARGET_DEVICES`: 

```c
// path: include/configs/rockchip-common.h
#define BOOT_TARGET_DEVICES(func) \
	BOOT_TARGET_MMC(func) \
	BOOT_TARGET_MTD(func) \
	BOOT_TARGET_RKNAND(func) \
	BOOT_TARGET_USB(func) \
	BOOT_TARGET_PXE(func) \
	BOOT_TARGET_DHCP(func)
```

`BOOTENV_DEV`定义如下：

```
#define BOOTENV_DEV_BLKDEV(devtypeu, devtypel, instance) \
	"bootcmd_" #devtypel #instance "=" \
		"setenv devnum " #instance "; " \
		"run " #devtypel "_boot\0"
```

逐级展开（只讨论mmc）:

```c
    BOOT_TARGET_DEVICES(BOOTENV_DEV)

    ->（这里就只展开mmc了，道理都一样）
        BOOT_TARGET_MMC(BOOTENV_DEV) \
    ->
    BOOTENV_DEV(MMC, mmc, 1) \
	BOOTENV_DEV(MMC, mmc, 0)"\0"
    ->
    "bootcmd_mmc1=setenv devnum 1;run mmc1_boot\0" \
    "bootcmd_mmc0=setenv devnum 0;run mmc0_boot\0" \
```




## Ref
https://blog.csdn.net/Andyshrk/article/details/89136721
https://juejin.cn/post/7202851843488088125