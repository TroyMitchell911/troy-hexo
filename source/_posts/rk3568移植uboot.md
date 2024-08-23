title: Rockchip 系列启动流程解读
date: '2024-08-23 12:57:43'
updated: '2024-08-23 12:57:45'
tags:
  - rockchip
categories:
  - kernel
  - rockchip
---

## Soc启动流程

Soc在上电之后，第一个执行的代码是芯片是`BootRom`，通常来说，SoC厂家都会做一个`ROM`在SoC的内部，这个`ROM`很小，里面固化了上电启动的代码（一经固化，永不能改，是芯片做的时候，做进去的）；这部分代码呢，我们管它叫做`BootROM`，也叫作`一级启动程序`。

BootRom需要做的事情：初始化系统，CPU的配置，关闭看门狗，初始化时钟，初始化一些外设（比如 `USB Controller`、`MMC Controller`，`Nand Controller`等）；

`BootROM`的代码除了去初始化硬件环境以外，还需要去外部存储器上面，将接下来可执行的程序读到内存来执行。

但此时`dram`还没有初始化完成，所以并不能直接读取程序到`dram`上执行，这时候芯片内部自带的`sram`就派上用场了，但`sram`造价高昂，所以通常内存容量较小，只能加载一小段程序到sram运行，这一小段程序只需要负责初始化`dram`。

初始化好`dram`之后将跳会到`BootRom`，`BootRom`再加载一段程序用以将`uboot`和`trust`复制到`dram`并运行。

上文提到了一个专业名词叫做`trust`，因为`RK3399`是`ARM64`，所以我们还需要编译·`ATF (ARM Trust Firmware)`，`ATF`主要负责在启动`uboot`之前把CPU从安全的`EL3`切换到`EL2`，然后跳转到`uboot`，并且在内核启动后负责启动其他的CPU。

### 开源方式

在开源方式中，初始化`dram`的程序叫做`TPL`; 将`uboot`加载到`dram`中的程序叫做`SPL`.

在`rockchip`移植好的uboot中开启以下选项即可编译出来`TPL / SPL`文件：

```bash
#
# SPL / TPL
#
CONFIG_SUPPORT_SPL=y
CONFIG_SUPPORT_TPL=y
CONFIG_SPL=y
# CONFIG_SPL_ADC_SUPPORT is not set
# CONFIG_SPL_DECOMP_HEADER is not set
CONFIG_SPL_BOARD_INIT=y
# CONFIG_SPL_BOOTROM_SUPPORT is not set
# CONFIG_SPL_RAW_IMAGE_SUPPORT is not set
# CONFIG_SPL_LEGACY_IMAGE_SUPPORT is not set
CONFIG_SPL_SYS_MALLOC_SIMPLE=y
# CONFIG_TPL_SYS_MALLOC_SIMPLE is not set
# CONFIG_SPL_STACK_R is not set
CONFIG_SPL_SEPARATE_BSS=y
# CONFIG_SPL_DISPLAY_PRINT is not set
CONFIG_SPL_SKIP_RELOCATE=y
CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_USE_SECTOR=y
CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_SECTOR=0x4000
CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_USE_PARTITION=y
CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_PARTITION=1
CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_PARTITION_NAME="uboot"
# CONFIG_SPL_CRC32_SUPPORT is not set
# CONFIG_SPL_MD5_SUPPORT is not set
# CONFIG_SPL_SHA1_SUPPORT is not set
CONFIG_SPL_SHA256_SUPPORT=y
# CONFIG_SPL_FIT_IMAGE_TINY is not set
# CONFIG_SPL_CPU_SUPPORT is not set
CONFIG_SPL_CRYPTO_SUPPORT=y
CONFIG_SPL_HASH_SUPPORT=y
# CONFIG_SPL_DMA_SUPPORT is not set
# CONFIG_SPL_ENV_SUPPORT is not set
# CONFIG_SPL_EXT_SUPPORT is not set
# CONFIG_SPL_FPGA_SUPPORT is not set
# CONFIG_SPL_I2C_SUPPORT is not set
CONFIG_SPL_MMC_WRITE=y
# CONFIG_SPL_MPC8XXX_INIT_DDR_SUPPORT is not set
CONFIG_SPL_MTD_SUPPORT=y
CONFIG_MTD_BLK_U_BOOT_OFFS=0x4000
CONFIG_SPL_MTD_WRITE=y
# CONFIG_SPL_MUSB_NEW_SUPPORT is not set
# CONFIG_SPL_NET_SUPPORT is not set
# CONFIG_SPL_NO_CPU_SUPPORT is not set
# CONFIG_SPL_NOR_SUPPORT is not set
# CONFIG_SPL_XIP_SUPPORT is not set
# CONFIG_SPL_ONENAND_SUPPORT is not set
# CONFIG_SPL_OS_BOOT is not set
# CONFIG_SPL_PCI_SUPPORT is not set
# CONFIG_SPL_PCH_SUPPORT is not set
# CONFIG_SPL_POST_MEM_SUPPORT is not set
# CONFIG_SPL_POWER_SUPPORT is not set
# CONFIG_SPL_PWM_SUPPORT is not set
# CONFIG_SPL_RAM_SUPPORT is not set
# CONFIG_SPL_RTC_SUPPORT is not set
# CONFIG_SPL_SATA_SUPPORT is not set
# CONFIG_SPL_RKNAND_SUPPORT is not set
# CONFIG_SPL_SPI_FLASH_TINY is not set
# CONFIG_SPL_SPI_FLASH_SFDP_SUPPORT is not set
# CONFIG_SPL_SPI_LOAD is not set
# CONFIG_SPL_USB_HOST_SUPPORT is not set
# CONFIG_SPL_USB_GADGET is not set
# CONFIG_SPL_YMODEM_SUPPORT is not set
CONFIG_SPL_ATF=y
# CONFIG_SPL_OPTEE_SUPPORT is not set
CONFIG_SPL_ATF_NO_PLATFORM_PARAM=y
# CONFIG_SPL_OPTEE is not set
CONFIG_SPL_AB=y
# CONFIG_SPL_LOAD_RKFW is not set
# CONFIG_SPL_KERNEL_BOOT is not set
CONFIG_TPL=y
# CONFIG_TPL_BOARD_INIT is not set
# CONFIG_TPL_NEEDS_SEPARATE_TEXT_BASE is not set
# CONFIG_TPL_NEEDS_SEPARATE_STACK is not set
# CONFIG_TPL_BOOTROM_SUPPORT is not set
# CONFIG_TPL_DRIVERS_MISC_SUPPORT is not set
# CONFIG_TPL_ENV_SUPPORT is not set
# CONFIG_TPL_I2C_SUPPORT is not set
CONFIG_TPL_TINY_FRAMEWORK=y
# CONFIG_TPL_MPC8XXX_INIT_DDR_SUPPORT is not set
# CONFIG_TPL_MMC_SUPPORT is not set
# CONFIG_TPL_NAND_SUPPORT is not set
CONFIG_TPL_SERIAL_SUPPORT=y
# CONFIG_TPL_SPI_FLASH_SUPPORT is not set
# CONFIG_TPL_SPI_SUPPORT is not set
```

编译出来的文件如下：

```bash
❯ ls spl
arch   cmd     disk     dts  fs       lib         u-boot-spl      u-boot-spl.dtb      u-boot-spl.lds  u-boot-spl-nodtb.bin
board  common  drivers  env  include  u-boot.cfg  u-boot-spl.bin  u-boot-spl-dtb.bin  u-boot-spl.map  u-boot-spl.sym
❯ ls tpl
arch   cmd     disk     dts  fs       lib         u-boot-spl.lds  u-boot-tpl.bin      u-boot-tpl.map        u-boot-tpl.sym
board  common  drivers  env  include  u-boot.cfg  u-boot-tpl      u-boot-tpl-dtb.bin  u-boot-tpl-nodtb.bin
```

### 闭源方式

在官方固件加载方式中，我们基于`Rockchip rkbin`官方给的`ddr.bin`、`miniloader.bin`来实现的: https://github.com/Caesar-github/rkbin

1.通过`tools/mkimage`将官方固件`ddr`, `miniloader`打包成`BootROM`程序可识别的、带有`ID Block header`的文件`idbloader.img`；

`ddr.bin`：等价于上面说的`TPL`，用于初始化`DDR`；
miniloader.bin：Rockchip修改的一个bootloader，等价于上面说的SPL，用于加载uboot；
这个文件打包出来实际上也是超过192KB的，因此也是分为二阶段执行的。

2. 通过tools/loaderimage工具将u-boot.bin打包成u-boot.img；其中u-boot.bin是由uboot源码编译生成；

补充说明：使用Rockchip miniloader的 idbloader 时，需要将u-boot.bin通过tools/loaderimage转换为可加载的miniloader格式。

3.使用Rockchip工具tools/trust_merge将bl31.bin打包成trust.img；其中bl31.bin由ATF源码编译生成；

补充说明：使用Rockchip miniloader的idbloader 时，需要将bl31.bin通过tools/trust_merge转换为可加载的miniloader格式。

## Ref

https://www.cnblogs.com/zyly/p/17380243.html
https://www.cnblogs.com/zyly/p/17389525.html#_label0
https://github.com/Caesar-github/docs/tree/master/Common/UBOOT