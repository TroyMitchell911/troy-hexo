title: 野火uboot使用extboot启动内核流程
date: '2024-08-26 15:43:00'
updated: '2024-08-27 11:15:29'
tags:
  - rockchip
  - rk3568
categories:
  - kernel
  - rockchip
---
查看野火`uboot`参数:

```bash
=> printenv bootcmd
bootcmd=run distro_bootcmd;boot_android ${devtype} ${devnum};boot_fit;bootrkp;
```

可以看到第一个命令是`distro_bootcmd`，事实上，野火的`extboot`也就是从这里启动的:

```bash
=> printenv distro_bootcmd 
distro_bootcmd=for target in ${boot_targets}; do run bootcmd_${target}; done
```

关于`distro_bootcmd`更详细的内容可以查看：https://blog.505218.xyz/2024/08/23/Rockchip-%E7%B3%BB%E5%88%97%E8%8A%AF%E7%89%87uboot-distro-cmd/

这里直接进入到`bootcmd_mmc0`，也就是从`emmc`启动，`sd`卡大同小异:

```bash
=> printenv bootcmd_mmc0
bootcmd_mmc0=setenv devnum 0; run mmc_boot
```

可以看到`bootcmd_mmc0`就是设置好`devnum`变量，然后执行了`mmc_boot`，这是为了将`sd`卡和`emmc`进行抽象，查看`mmc_boot`:

```bash
=> printenv mmc_boot
mmc_boot=if mmc dev ${devnum}; then setenv devtype mmc; run scan_dev_for_boot_part; fi
```

在`mmc_boot`阶段，设置了变量`devtype`为`mmc`，然后执行`scan_dev_for_boot_part`:

```bash
=> printenv scan_dev_for_boot_part
scan_dev_for_boot_part=part list ${devtype} ${devnum} -bootable devplist; env exists devplist || setenv devplist 1; for distro_bootpart in ${devplist};
 do if fstype ${devtype} ${devnum}:${distro_bootpart} bootfstype; then run scan_dev_for_boot; fi; done
```

## scan_dev_for_boot_part

在`scan_dev_for_boot_part`阶段就比较复杂了，在uboot中不方便查看，我们转到头文件查看：

```c
// path: ./include/config_distro_bootcmd.h
	"scan_dev_for_boot_part="                                         \
		"part list ${devtype} ${devnum} -bootable devplist; "     \
		"env exists devplist || setenv devplist 1; "              \
		"for distro_bootpart in ${devplist}; do "                 \
			"if fstype ${devtype} "                           \
					"${devnum}:${distro_bootpart} "   \
					"bootfstype; then "               \
				"run scan_dev_for_boot; "                 \
			"fi; "                                            \
		"done\0"                                                  \
```

诸行分析：
    - 列出`${devtype} ${devnum} `下所有`bootable`的分区并且将其放到`devplist`变量中
    - 判断`devplist`是否存在，也就是上一步是否成功，如果不成功设置为`1`
    - 使用`distro_bootpart`变量遍历`devplist`，假如`devplist`是`2`,那么`distro_bootpart`会是`0,1,2`
    - 在`for`循环中输出`${devtype} ${devnum}:${distro_bootpart}`的文件类型给`bootfstype`变量
    - 运行`scan_dev_for_boot`

## scan_dev_for_boot

查看`scan_dev_for_boot`:

```c
// path: ./include/config_distro_bootcmd.h
	"scan_dev_for_boot="                                              \
		"echo Scanning ${devtype} "                               \
				"${devnum}:${distro_bootpart}...; "       \
		"for prefix in ${boot_prefixes}; do "                     \
			"run scan_dev_for_scripts; "                      \
			"run scan_dev_for_extlinux; "                     \
		"done;"                                                   \
		SCAN_DEV_FOR_EFI                                          \
		"\0"                                                      \
```

这里其实就是打印一个调试信息，然后遍历`boot_prefixes`去运行两个命令。

查看`boot_prefixes`:

```bash
=> printenv boot_prefixes 
boot_prefixes=/ /boot/
```

就是一个`boot`的前缀，一个是`/`一个是`/boot/`。

## scan_dev_for_scripts

查看`scan_dev_for_scripts`:

```c
// path: ./include/config_distro_bootcmd.h
	"scan_dev_for_scripts="                                           \
		"for script in ${boot_scripts}; do "                      \
			"if test -e ${devtype} "                          \
					"${devnum}:${distro_bootpart} "   \
					"${prefix}${script}; then "       \
				"echo Found U-Boot script "               \
					"${prefix}${script}; "            \
				"run boot_a_script; "                     \
				"echo SCRIPT FAILED: continuing...; "     \
			"fi; "                                            \
		"done\0"                                                  \
```

其中涉及到一个变量为`boot_scripts`:

```bash
=> printenv boot_scripts 
boot_scripts=boot.scr.uimg boot.scr
```

诸行分析`scan_dev_for_scripts`：
    - 使用`script`变量遍历`boot_scripts`
    - 判断`${devnum}:${distro_bootpart} ${prefix}${script}` 文件是否存在
    - 如果存在运行`boot_a_script`

查看`boot_a_script`:

```bash
=> printenv boot_a_script 
boot_a_script=load ${devtype} ${devnum}:${distro_bootpart} ${scriptaddr} ${prefix}${script}; source ${scriptaddr}
```

将刚才的`script`文件加载到`scriptaddr`，并且运行该脚本，`source`命令就是运行脚本。

```bash
=> printenv scriptaddr
scriptaddr=0x00c00000
```

## scan_dev_for_extlinux

查看`scan_dev_for_extlinux`:

```c
// path: ./include/config_distro_bootcmd.h
	"scan_dev_for_extlinux="                                          \
		"if test -e ${devtype} "                                  \
				"${devnum}:${distro_bootpart} "           \
				"${prefix}extlinux/extlinux.conf; then "  \
			"echo Found ${prefix}extlinux/extlinux.conf; "    \
			"run boot_extlinux; "                             \
			"echo SCRIPT FAILED: continuing...; "             \
		"fi\0"                                                    \
```

这里就是判断`${devtype} ${devnum}:${distro_bootpart} ${prefix}extlinux/extlinux.conf`文件是否存在，如果存在的话，就去运行`boot_extlinux`。

## boot_extlinux

查看`boot_extlinux`:

```bash
=> printenv boot_extlinux 
boot_extlinux=sysboot ${devtype} ${devnum}:${distro_bootpart} any ${scriptaddr} ${prefix}extlinux/extlinux.conf
```

## Ref

https://doc.embedfire.com/lubancat-mp157/build_and_deploy/zh/latest/building_image/use_uboot/use_uboot.html