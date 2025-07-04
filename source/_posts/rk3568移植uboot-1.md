title: rk3568移植uboot
date: '2024-08-26 10:52:09'
updated: '2024-11-06 13:39:48'
---
# Env

Systemt: Ubuntu 22.04

## Content

首先克隆仓库：

```bash
❯ git clone --depth=1  https://github.com/Caesar-github/u-boot
# 这是rkbin仓库，至于为什么克隆见https://blog.troy-y.org/2024/08/23/rk3568%E7%A7%BB%E6%A4%8Duboot/
❯ git clone --depth=1 git@github.com:Caesar-github/rkbin.git
```

执行以下命令：

```bash
❯ cd uboot && ./make.sh rk3568
```
生成以下文件：

```bash
❯ ls rk356x_spl_loader_v1.13.112.bin
rk356x_spl_loader_v1.13.112.bin
❯ ls uboot.img
uboot.img
```

其中`rk356x_spl_loader_v1.13.112.bin`是由rkbin仓库的`ddr init.bin`和`miniloader`合并成的，了解原理并手动合成可以看https://blog.troy-y.org/2024/08/23/rk3568%E7%A7%BB%E6%A4%8Duboot/

`uboot.img`则是由`fit/u-boot.itb`拷贝来的，`fit/u-boot.itb`是一个`FIT`格式的镜像文件，由`fit/u-boot.its`指导生成，观察`fit/u-boot.its`文件如下：

```bash

/*
 * Copyright (C) 2020 Rockchip Electronic Co.,Ltd
 *
 * Simple U-boot fit source file containing ATF/OP-TEE/U-Boot/dtb/MCU
 */

/dts-v1/;

/ {
	description = "FIT Image with ATF/OP-TEE/U-Boot/MCU";
	#address-cells = <1>;

	images {

		uboot {
			description = "U-Boot";
			data = /incbin/("u-boot-nodtb.bin");
			type = "standalone";
			arch = "arm64";
			os = "U-Boot";
			compression = "none";
			load = <0x00a00000>;
			hash {
				algo = "sha256";
			};
		};
		atf-1 {
			description = "ARM Trusted Firmware";
			data = /incbin/("./bl31_0x00040000.bin");
			type = "firmware";
			arch = "arm64";
			os = "arm-trusted-firmware";
			compression = "none";
			load = <0x00040000>;
			hash {
				algo = "sha256";
			};
		};
		atf-2 {
			description = "ARM Trusted Firmware";
			data = /incbin/("./bl31_0xfdcc1000.bin");
			type = "firmware";
			arch = "arm64";
			os = "arm-trusted-firmware";
			compression = "none";
			load = <0xfdcc1000>;
			hash {
				algo = "sha256";
			};
		};
		atf-3 {
			description = "ARM Trusted Firmware";
			data = /incbin/("./bl31_0x0006a000.bin");
			type = "firmware";
			arch = "arm64";
			os = "arm-trusted-firmware";
			compression = "none";
			load = <0x0006a000>;
			hash {
				algo = "sha256";
			};
		};
		atf-4 {
			description = "ARM Trusted Firmware";
			data = /incbin/("./bl31_0xfdcd0000.bin");
			type = "firmware";
			arch = "arm64";
			os = "arm-trusted-firmware";
			compression = "none";
			load = <0xfdcd0000>;
			hash {
				algo = "sha256";
			};
		};
		atf-5 {
			description = "ARM Trusted Firmware";
			data = /incbin/("./bl31_0xfdcce000.bin");
			type = "firmware";
			arch = "arm64";
			os = "arm-trusted-firmware";
			compression = "none";
			load = <0xfdcce000>;
			hash {
				algo = "sha256";
			};
		};
		atf-6 {
			description = "ARM Trusted Firmware";
			data = /incbin/("./bl31_0x00068000.bin");
			type = "firmware";
			arch = "arm64";
			os = "arm-trusted-firmware";
			compression = "none";
			load = <0x00068000>;
			hash {
				algo = "sha256";
			};
		};
		optee {
			description = "OP-TEE";
			data = /incbin/("tee.bin");
			type = "firmware";
			arch = "arm64";
			os = "op-tee";
			compression = "none";
			
			load = <0x8400000>;
			hash {
				algo = "sha256";
			};
		};
		fdt {
			description = "U-Boot dtb";
			data = /incbin/("./u-boot.dtb");
			type = "flat_dt";
			arch = "arm64";
			compression = "none";
			hash {
				algo = "sha256";
			};
		};
	};

	configurations {
		default = "conf";
		conf {
			description = "rk3568-evb";
			rollback-index = <0x0>;
			firmware = "atf-1";
			loadables = "uboot", "atf-2", "atf-3", "atf-4", "atf-5", "atf-6", "optee";
			
			fdt = "fdt";
			signature {
				algo = "sha256,rsa2048";
				
				key-name-hint = "dev";
				sign-images = "fdt", "firmware", "loadables";
			};
		};
	};
};
```

可以看到这里面将`bl31`文件分成了很多块和`u-boot-nodtb.bin`还有`u-boot.dtb`放入了`u-boot.itb`，`bl31`文件是一个`trust`文件，也就是`ATF (Arm trust firmware)`，`ATF `主要负责在启动`uboot`之前把`CPU`从安全的`EL3`切换到 `EL2`，然后跳转到`uboot`，并且在内核启动后负责启动其他的CPU。

## Ref

https://www.cnblogs.com/zyly/p/17389525.html#_label0
