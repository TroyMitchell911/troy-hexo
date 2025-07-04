title: 野火uboot使用extboot启动内核流程
date: '2024-08-26 15:43:00'
updated: '2024-08-27 13:30:53'
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

关于`distro_bootcmd`更详细的内容可以查看：https://blog.troy-y.org/2024/08/23/Rockchip-%E7%B3%BB%E5%88%97%E8%8A%AF%E7%89%87uboot-distro-cmd/

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

### 查看野火boot分区下脚本文件

```bash
root@lubancat:/boot# ls *.scr
boot.scr
```

这里就不分析该内容了，该文件内容如下：

```
if test -e ${devtype} ${devnum}:${distro_bootpart} /uEnv/uEnv.txt; then  
 
    echo [boot.cmd] load ${devtype} ${devnum}:${distro_bootpart} ${env_addr_r} //
uEnv/uEnv.txt ...; 
    load ${devtype} ${devnum}:${distro_bootpart} ${env_addr_r} /uEnv/uEnv.txt; 
 
    echo [boot.cmd] Importing environment from ${devtype} ... 
    env import -t ${env_addr_r} 0x8000 
 
    setenv bootargs ${bootargs} root=/dev/mmcblk${devnum}p3 ${cmdline} 
    printenv bootargs 
 
    echo [boot.cmd] load ${devtype} ${devnum}:${distro_bootpart} ${ramdisk_addr__
r} /initrd-${uname_r} ... 
    load ${devtype} ${devnum}:${distro_bootpart} ${ramdisk_addr_r} /initrd-${unaa
me_r} 
 
    echo [boot.cmd] loading ${devtype} ${devnum}:${distro_bootpart} ${kernel_addd
r_r} /Image-${uname_r} ... 
    load ${devtype} ${devnum}:${distro_bootpart} ${kernel_addr_r} /Image-${unamee
_r}

    echo [boot.cmd] loading default rk-kernel.dtb
    load ${devtype} ${devnum}:${distro_bootpart} ${fdt_addr_r} /rk-kernel.dtb

    fdt addr  ${fdt_addr_r}
    fdt set /chosen bootargs

    echo [boot.cmd] dtoverlay from /uEnv/uEnv.txt
    setenv dev_bootpart ${devnum}:${distro_bootpart}
    dtfile ${fdt_addr_r} ${fdt_over_addr}  /uEnv/uEnv.txt ${env_addr_r}

    echo [boot.cmd] [${devtype} ${devnum}:${distro_bootpart}] ...
    echo [boot.cmd] [booti] ...
    booti ${kernel_addr_r} ${ramdisk_addr_r} ${fdt_addr_r}
fi

echo [boot.cmd] run boot.cmd scripts failed ...;

# Recompile with:
# mkimage -C none -A arm -T script -d /boot/boot.cmd /boot/boot.scr
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

`boot_extlinux`中实际调用了`sysboot`命令，`sysboot` 是一个 `U-Boot` 命令，用于从特定设备和文件系统加载并启动 `Syslinux` 引导文件。`Syslinux` 是一个轻量级的引导加载程序，通常用于引导 `Linux` 内核。

### sysboot

命令格式如下：

```bash
sysboot [-p] <interface> <dev[:part]> <ext2|fat|any> [addr] [filename]
```

- -p: 可选参数，表示使用 "pxelinux" 格式的配置文件。
- <interface>: 设备接口类型，例如 mmc、usb、scsi、eth 等。
- <dev[:part]>: 设备号和可选的分区号。例如，0:1 表示设备 0 的第 1 分区。
- <ext2|fat|any>: 文件系统类型，可以是 ext2、fat 或 any。其中 any 会自动检测文件系统类型。
- [addr]: 可选参数，表示将文件加载到内存中的地址。
- [filename]: 可选参数，要加载和解析的 Syslinux 配置文件名，通常是 syslinux.cfg 或 extlinux.conf。

所以`boot_extlinux`中就是将`${devtype} ${devnum}:${distro_bootpart}`分区中的`${prefix}extlinux/extlinux.conf`文件加载到`{scriptaddr}`去用`sysboot`运行，该分区的文件系统类型设置为`any`意思就是让`uboot`自动检测文件系统类型。

### cfg文件


Syslinux 配置文件（通常命名为 syslinux.cfg 或 extlinux.conf）用于指定启动项和相关参数。这个文件的格式比较简单，通常包含启动菜单和内核启动选项。以下是一个典型的 syslinux.cfg 配置文件示例：

```
DEFAULT linux
PROMPT 0
TIMEOUT 50

LABEL linux
    KERNEL /boot/vmlinuz
    APPEND root=/dev/sda1 ro quiet
    INITRD /boot/initrd.img
```

- DEFAULT: 指定默认启动项的标签（如 LABEL 中定义的名称）。
- PROMPT: 设置是否显示引导菜单。0 表示不显示，直接启动默认项；1 表示显示引导菜单。
- TIMEOUT: 设置等待时间（以1/10秒为单位）。例如，50 表示等待5秒，然后启动默认项。
- LABEL: 定义一个启动项，并给它一个名称（如 linux）。
	- KERNEL: 指定要启动的内核文件路径。
	- APPEND: 向内核传递的启动参数。例如，root=/dev/sda1 ro quiet 表示根文件系统在 /- dev/sda1，只读挂载，并且启动时保持静默模式。
	- INITRD: 指定初始 RAM 磁盘映像（initrd）文件的路径，通常用于加载必要的驱动程序和文件系统支持。

还可以使用**更高级的选项**：

- SAY: 显示消息给用户。
- MENU DEFAULT: 指定一个菜单项为默认选项。
- FALLBACK: 在默认启动项失败时，指定一个后备启动项。
- MENU LABEL: 给每个启动项设置一个显示在菜单中的名称。
- single: 在 APPEND 中添加 single 参数将 Linux 以单用户模式启动，通常用于故障排查。

```
DEFAULT linux
PROMPT 1
TIMEOUT 100

SAY Hello! Welcome to the Syslinux bootloader.

LABEL linux
    MENU LABEL Start Linux
    KERNEL /boot/vmlinuz
    APPEND root=/dev/sda1 ro quiet
    INITRD /boot/initrd.img
    MENU DEFAULT

LABEL fallback
    MENU LABEL Fallback Linux
    KERNEL /boot/vmlinuz
    APPEND root=/dev/sda2 ro single
    INITRD /boot/initrd.img
    FALLBACK
```

这里需要重点说明的是设备树相关的问题，如果想指定设备树，需要以这样的格式进行指定：

```
FDT /your-device-tree.dtb
```

或者不明确指定设备树，但指定设备树所存在的文件夹，可以用`devicetreedir`或者`fdtdir`去指定。

```
devicetreedir /
```

或者是

```
fdtdir/
```

如果是使用指定设备树文件夹，不明确指定设备树的方式，`uboot`会使用以下策略来查找设备树：

- 自动选择: U-Boot 可能会自动选择与内核文件同名或类似名称的 .dtb 文件。如果内核名是 Image-4.19.232，那么它可能会尝试查找 Image-4.19.232.dtb 或其他类似名称的设备树文件。
- 在 U-Boot 中，可以通过环境变量（如 fdtfile）指定设备树文件的路径。如果没有在 extlinux.conf 中明确指定设备树，U-Boot 可能会使用这些环境变量中定义的路径。

### 查看野火boot分区下extlinux.conf

```bash
root@lubancat:/boot# cat extlinux/extlinux.conf 
label kernel-4.19.232
        kernel /Image-4.19.232
        devicetreedir /
        append  root=/dev/mmcblk0p3 earlyprintk console=ttyFIQ0 console=tty1 consoleblank=0 loglevel=7 rootwait rw rootfstype=ext4 cgroup_enable=cpuset
 cgroup_memory=1 cgroup_enable=memory swapaccount=1 switolb=1 coherent_pool=1m
```

可以看到野火的`conf`文件并没有明确指定设备树，而是指定了一个设备树文件夹，查看设备树文件：

```bash
root@lubancat:/boot# ls *.dtb
rk-kernel.dtb
root@lubancat:/boot# ls Image-4.19.232 
Image-4.19.232
```

可以看到并没有与内核同名的设备树文件，那么一定是使用了`fdtfile`变量来指定设备树。

进入`uboot`查看`fdtfile`:

```bash
=> printenv fdtfile 
fdtfile=rk-kernel.dtb
```

## Ref

https://doc.embedfire.com/lubancat-mp157/build_and_deploy/zh/latest/building_image/use_uboot/use_uboot.html