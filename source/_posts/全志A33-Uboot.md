title: '[全志A33-Vstar]Uboot'
date: '2024-09-13 10:58:32'
updated: '2024-09-20 22:39:38'
tags:
  - uboot
categories:
  - uboot
---
## Env

cpu: allwinner a33
board: vstar
host: ubuntu 22.04

需要安装交叉编译工具链：

```bash
❯ sudo apt install gcc-arm-linux-gnueabihf
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

## uboot-sunxi仓库试错

**该章节并没有成功，所以如果想学习可以看看，只是想移植uboot就可以跳到下一个章节**

要移植uboot首先想到的肯定是全志的uboot仓库，先拉下来代码：

```bash
❯ git clone https://github.com/linux-sunxi/u-boot-sunxi 
❯ cd u-boot-sunxi 
```

使用一个A33芯片的板子的config进行配置并编译，经过测试, A33-OLinuXino_defconfig这个配置文件没支持emmc，只支持了sd，这里选择sinlinx的config，与vstar开发板相近：

```bash
❯ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- Sinlinx_SinA33_defconfig
❯ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16
```

编译遇到如下错误，看起来很熟悉，好像当年搞f1c200s的时候也遇见了这个错误...

给lichee pi的仓库提交了pr，但已经过去了一年多，也没有理我...

该错误可通过该pr修复：[我是PR](https://github.com/Lichee-Pi/u-boot/pull/7)

```bash
usr/bin/ld: scripts/dtc/dtc-parser.tab.o:(.bss+0x10): multiple definition of `yylloc'; scripts/dtc/dtc-lexer.lex.o:(.bss+0x0): first defined here
  UPD     include/generated/version_autogenerated.h
  CC      lib/asm-offsets.s
  CC      arch/arm/lib/asm-offsets.s
collect2: error: ld returned 1 exit status
make[2]: *** [scripts/Makefile.host:106：scripts/dtc/dtc] 错误 1
make[2]: *** 正在等待未完成的任务....
```

继续编译，到了最后阶段却遇到了如下错误：

```bash
  MKSUNXI spl/sunxi-spl.bin
  BINMAN  u-boot-sunxi-with-spl.bin
binman: No module named _libfdt
make: *** [Makefile:1348：u-boot-sunxi-with-spl.bin] 错误 1
```

检查python是否有这个module:

```bash
❯ python -m pip list | grep pylibfdt
pylibfdt                           1.7.0.post1
```

这时候已经开始怀疑是这个module的版本问题了，在[这里](https://pypi.org/project/pylibfdt/#history)找到了历史版本信息，逐个尝试安装还是不行...

最终在[这里](https://lore.kernel.org/buildroot/bug-11706-163@https.bugs.busybox.net%2F/T/)找到了问题的原因，是python版本的问题。

在[这里](https://blog.csdn.net/qq_34752070/article/details/125182978)找到了问题的解决方案，重新链接后解决问题。

连接开发板串口0之后通过进入fel启动uboot：

```bash
❯ git clone git@github.com:u-boot/u-boot.git
❯ sudo sunxi-fel uboot ./u-boot-sunxi-with-spl.bin
```

之后发现emmc和sd卡都无法使用...

```bash
=> mmc dev 0
Card did not respond to voltage select!
=> mmc dev 1
Card did not respond to voltage select!
=> mmc dev 0
MMC: no card present
=> mmc dev 1
Card did not respond to voltage select!
```

**fuck you allwinner! mainline start!**

## mainline

### 运行

拉取uboot主线代码：

```bash
❯ git clone git@github.com:u-boot/u-boot.git 
❯ cd u-boot
```

依旧使用之前的Sinlinx的配置：

```bash
❯ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- Sinlinx_SinA33_defconfig
❯ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16
```

这次发现日志到MMC就结束了：

```bash
U-Boot 2024.10-rc4-g5f0449324136-dirty (Sep 14 2024 - 15:47:46 +0800) Allwinner Technology

CPU:   Allwinner A33 (SUN8I 1667)
Model: Sinlinx SinA33
DRAM:  512 MiB
Core:  64 devices, 21 uclasses, devicetree: separate
WDT:   Not starting watchdog@1c20ca0
MMC: 
```

这里一开始一直怀疑uboot卡死了，排查一段时间后发现其实并不是。

可以观察到`Model: Sinlinx SinA33`这一行，按找这个查找对应设备树：

```bash
❯ find -name "*.dts*" -exec grep -n "Sinlinx SinA33" {} +
./arch/arm/dts/sun8i-a33-sinlinx-sina33.dts:53:	model = "Sinlinx SinA33";
./dts/upstream/src/arm/allwinner/sun8i-a33-sinlinx-sina33.dts:53:	model = "Sinlinx SinA33";
```

查到两个dts，查询dtb:

```bash
❯ ls ./arch/arm/dts/sun8i-a33-sinlinx-sina33.dtb
./arch/arm/dts/sun8i-a33-sinlinx-sina33.dtb
❯ ls ./dts/upstream/src/arm/allwinner/sun8i-a33-sinlinx-sina33.dtb
ls: 无法访问 './dts/upstream/src/arm/allwinner/sun8i-a33-sinlinx-sina33.dtb': 没有那个文件或目录
```

可以确定就是`./arch/arm/dts/sun8i-a33-sinlinx-sina33.dtb`这个文件，查询其对应的dts可以发现指定的就是uart0：

```bash
	aliases {
		serial0 = &uart0;
	};

	chosen {
		stdout-path = "serial0:115200n8";
	};
```

查看uart0节点及其对应的gpio：

```bash
// ./arch/arm/dts/sun8i-a33-sinlinx-sina33.dts
&uart0 {
	pinctrl-names = "default";
	pinctrl-0 = <&uart0_pb_pins>;
	status = "okay";
};

//./arch/arm/dts/sun8i-a33.dtsi
&pio {
	compatible = "allwinner,sun8i-a33-pinctrl";
	interrupts = <GIC_SPI 15 IRQ_TYPE_LEVEL_HIGH>,
		     <GIC_SPI 17 IRQ_TYPE_LEVEL_HIGH>;

	uart0_pb_pins: uart0-pb-pins {
		pins = "PB0", "PB1";
		function = "uart0";
	};
};
```

这里指定了uart0的引脚使用PB0作tx，PB1作rx，查看原理图这两个引脚原来在是uart2,将uart2的引脚复用成uart0了

[![2024-09-14-16-44-32.png](https://i.postimg.cc/dV2cwPzZ/2024-09-14-16-44-32.png)](https://postimg.cc/TK34JS2T)

那么这样就可以分析出来原因了，由于chosen指定的输出端口是uart0,所以在没有uart2的gpio复用成uart0的功能之前，就由uart0默认的对应引脚PF2 PF4输出，复用之后就使用了uart2的gpio作uart0。

连接上uart2之后果然可以看到完整log，但这里还是修改一下吧。

### 串口修改

在修改之前，先复制一份dts和配置文件，作为我们自己的vstar的：

```bash
❯ cp ./arch/arm/dts/sun8i-a33-sinlinx-sina33.dts ./arch/arm/dts/sun8i-a33-vstar.dts
❯ cp ./configs/Sinlinx_SinA33_defconfig ./configs/VStar_A33_defconfig
```

修改Makefile支持dts:

```bash
diff --git a/arch/arm/dts/Makefile b/arch/arm/dts/Makefile
index 2d931c23fc8..1cf745f8027 100644
--- a/arch/arm/dts/Makefile
+++ b/arch/arm/dts/Makefile
@@ -607,6 +607,7 @@ dtb-$(CONFIG_MACH_SUN8I_A33) += \
        sun8i-a33-olinuxino.dtb \
        sun8i-a33-q8-tablet.dtb \
        sun8i-a33-sinlinx-sina33.dtb \
+       sun8i-a33-vstar.dtb \
        sun8i-r16-bananapi-m2m.dtb \
        sun8i-r16-nintendo-nes-classic.dtb \
        sun8i-r16-nintendo-super-nes-classic.dtb \
```

修改config选择生成镜像内的设备树文件：

```bash
❯ diff configs/VStar_A33_defconfig configs/Sinlinx_SinA33_defconfig
3c3
< CONFIG_DEFAULT_DEVICE_TREE="sun8i-a33-vstar"
---
> CONFIG_DEFAULT_DEVICE_TREE="sun8i-a33-sinlinx-sina33"
```

使用自己的配置文件编译一次：

```bash
❯ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- VStar_A33_defconfig
#
# configuration written to .config
#
❯ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16
```

编译成功，现在来修改dts以让他全部走uart2,这样uart0就没有任何输出了。

不然明明连接的uart2,结果uart0显示一段感觉怪怪的。

```bash
diff --git a/arch/arm/dts/sun8i-a33-vstar.dts b/arch/arm/dts/sun8i-a33-vstar.dts
index d54a067fc76..23d73f4f62f 100644
--- a/arch/arm/dts/sun8i-a33-vstar.dts
+++ b/arch/arm/dts/sun8i-a33-vstar.dts
@@ -54,11 +54,11 @@
 	compatible = "sinlinx,sina33", "allwinner,sun8i-a33";
 
 	aliases {
-		serial0 = &uart0;
+		serial2 = &uart2;
 	};
 
 	chosen {
-		stdout-path = "serial0:115200n8";
+		stdout-path = "serial2:115200n8";
 	};
 
 	panel {
@@ -261,6 +261,12 @@
 &uart0 {
 	pinctrl-names = "default";
 	pinctrl-0 = <&uart0_pb_pins>;
+	status = "disabled";
+};
+
+&uart0 {
+	pinctrl-names = "default";
+	pinctrl-0 = <&uart2_pins>;
 	status = "okay";
 };
 
@@ -273,3 +279,10 @@
 	status = "okay";
 	usb1_vbus-supply = <&reg_vcc5v0>; /* USB1 VBUS is always on */
 };
+
+&pio {
+	uart2_pins: uart2-pins {
+		pins = "PB0", "PB1";
+		function = "uart2";
+	};
+};
diff --git a/configs/VStar_A33_defconfig b/configs/VStar_A33_defconfig
index a428e68725f..c6ea136733c 100644
--- a/configs/VStar_A33_defconfig
+++ b/configs/VStar_A33_defconfig
@@ -15,6 +15,7 @@ CONFIG_VIDEO_LCD_BL_PWM="PH0"
 CONFIG_CMD_DFU=y
 CONFIG_DFU_RAM=y
 CONFIG_FASTBOOT_CMD_OEM_FORMAT=y
+CONFIG_CONS_INDEX=3
 CONFIG_USB_EHCI_HCD=y
 CONFIG_USB_OHCI_HCD=y
 CONFIG_USB_MUSB_GADGET=y
```

### 网口使能

查看原理图可以发现网口使用了rtl8152芯片，这个芯片通过usb通讯，所以dts中根本就不用改动，config中使能该驱动就可以：

[![2024-09-14-17-12-17.png](https://i.postimg.cc/YSBKsvdD/2024-09-14-17-12-17.png)](https://postimg.cc/w12Gtj8D)

在menuconfig中使能该驱动，以下是使能该驱动之后config文件的变更：

```bash
❯ diff defconfig configs/VStar_A33_defconfig
22,23d21
< CONFIG_USB_HOST_ETHER=y
< CONFIG_USB_ETHER_RTL8152=y
```

重新编译下载，使用dhcp获取ip:

```bash
=> dhcp
musb-hdrc: peripheral reset irq lost!
high speed config #2: 2 mA, Ethernet Gadget, using RNDIS
USB RNDIS network up!
BOOTP broadcast 1
BOOTP broadcast 2
BOOTP broadcast 3
BOOTP broadcast 4
```

什么鬼，看起来走了另一个网口，但是哪来的网口，唯一怀疑的就是usb口的otg模式，rndis协议模拟网口了。

查看设备树果然有这么一个节点，将其改成disabled：

```bash
&usb_otg {
	dr_mode = "peripheral";
	status = "disabled";
};
```

重新编译下载发现dhcp显示没有网口，那么看来网口无法识别，注意到uboot启动时有一行显示mac地址无效：

```bash
Error: r8152_eth No valid MAC address found.
```

在[这里](https://blog.csdn.net/weixin_43479963/article/details/107073502)找到了资料，什么！？ mac还需要购买后通过专用工具烧写，去你的吧，这篇博文就从驱动修改入手直接写入了mac地址，但他那是linux的驱动，跟我们这里还是查很多的，我们这里就没有write函数，自己实现起来费力不讨好，而且rtl8152的register的datasheet没有找到，所以这里我们从read入手。

首先在read函数中加入打印函数，查看一下获取到的mac地址，果然全是0。

那么很简单，既然读不到mac，那么我就造一个：

```bash
diff --git a/drivers/usb/eth/r8152.c b/drivers/usb/eth/r8152.c
index e3f20e08c33..672b8dac8a6 100644
--- a/drivers/usb/eth/r8152.c
+++ b/drivers/usb/eth/r8152.c
@@ -625,10 +625,14 @@ static int r8152_read_mac(struct r8152 *tp, unsigned char *macaddr)
        int ret;
        unsigned char enetaddr[8] = {0};
 
        ret = pla_ocp_read(tp, PLA_IDR, 8, enetaddr);
        if (ret < 0)
                return ret;
 
+       if(is_zero_ethaddr(enetaddr) || !is_valid_ethaddr(enetaddr))
+               net_random_ethaddr(enetaddr);
+
        memcpy(macaddr, enetaddr, ETH_ALEN);
        return 0;
 }
(END)
```

添加的内容就是判断如果mac地址无效的情况下伪造一个fake-mac，这样就可以了。

重新编译烧写，果然没有那一行报错了，通过dhcp命令动态获取ip:

```bash
=> dhcp
BOOTP broadcast 1
DHCP client bound to address 192.168.230.34 (36 ms)
*** Warning: no boot file name; using 'C0A8E622.img'
Using r8152_eth device
TFTP from server 192.168.230.1; our IP address is 192.168.230.34
Filename 'C0A8E622.img'.
Load address: 0x42000000
Loading: *
TFTP error: 'File not found' (1)
Not retrying...
```

这次果然成功获取到ip了，这下可以使用tftp网络获取内核了，调试方便很多。

但是这里报了一些找不到文件的错误，这是uboot机制，设置一个环境变量就可以解决了

```bash
=> setenv autoload no
```

由于是fel模式，烧写进入的是ram，掉电会丢失，所以saveenv保存不了参数，那么直接把autoload环境变量做到uboot源码里面也是可以的。

只需要修改`include/configs/sunxi-common.h`文件即可：

```bash
diff --git a/include/configs/sunxi-common.h b/include/configs/sunxi-common.h
index b29a25d5617..fefc2e291eb 100644
--- a/include/configs/sunxi-common.h
+++ b/include/configs/sunxi-common.h
@@ -297,6 +297,7 @@
        "uuid_gpt_system=" UUID_GPT_SYSTEM "\0" \
        "partitions=" PARTS_DEFAULT "\0" \
        BOOTCMD_SUNXI_COMPAT \
-       BOOTENV
+       BOOTENV \
+       "autoload=no"
 
 #endif /* _SUNXI_COMMON_CONFIG_H */
```




