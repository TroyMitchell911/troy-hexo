---
title: smt-august-1-u2w
date: 2025-08-01 15:27:58
tags:
  - smt
  - update
  - summary
---
## 汇报分享

### ASoC学习与upstream

ASoC--ALSA System on Chip ，是建立在标准ALSA驱动层上，为了更好地支持嵌入式处理器和移动设备中的音频Codec的一套软件体系。在ASoc出现之前，内核对于SoC中的音频已经有部分的支持，不过会有一些局限性：

Codec驱动与SoC CPU的底层耦合过于紧密，这种不理想会导致代码的重复.

例如，仅是wm8731的驱动，当时Linux中有分别针对4个平台的驱动代码。

- 音频事件没有标准的方法来通知用户，例如耳机、麦克风的插拔和检测，这些事件在移动设备中是非常普通的，而且通常都需要特定于机器的代码进行重新对音频路劲进行配置。

- 当进行播放或录音时，驱动会让整个codec处于上电状态，这对于PC没问题，但对于移动设备来说，这意味着浪费大量的电量。同时也不支持通过改变过取样频率和偏置电流来达到省电的目的。

ASoC就是在这样的背景下诞生的，为了支持移动设备中的音频，ASoC将Codec驱动与SoC CPU解耦，通过标准ALSA接口向上提供统一的音频服务。这样，用户只需要安装ASoC驱动，就可以使用各种Codec，而不需要针对不同SoC平台编写不同驱动。同时，ASoC还提供了标准的音频事件通知机制，使得用户可以方便地进行音频配置。

#### ASoC的分层

想使用ASoC驱动框架编写驱动需要了解三个层次：
  - Platform: 这一层是平台层，所有与平台(SoC)相关的代码都属于这一层。比如DAI，PCM等。
  - Codec: 这一层是声卡层，一般来说，声卡厂家会提供该层的驱动。比如wm8731.
  - Machine: 这一层属于胶水层，起到连接Platform和Codec的作用。比如指定CPU的DAI是哪一个，指定Codec的是哪一个，然后打包成一个声卡（Card）设备在系统中体现出来。

#### 现代ASoC驱动的编写

##### DAI

无论什么时候，一个未被支持的设备想支持ASoC,首先需要编写DAI。

因为DAI是一个声卡设备的核心，是底层接口。

它决定了SoC和Codec之间的通信协议, 是最底层的支撑。

想实现DAI，就要了解dai_ops这个数据结构，这是一个操作接口，提供给上层。

```
static const struct snd_soc_dai_ops spacemit_i2s_dai_ops = {
	.probe = spacemit_i2s_dai_probe,
	.startup = spacemit_i2s_startup,
	.hw_params = spacemit_i2s_hw_params,
	.set_sysclk = spacemit_i2s_set_sysclk,
	.set_fmt = spacemit_i2s_set_fmt,
	.trigger = spacemit_i2s_trigger,
};
```

- probe: dai注册成功后会调用，负责一些初始化, 在k1中是设置了dma的参数和复位了i2s
- startup: 打开声卡设备时的调用，主要是电源/时钟等使能
- hw_params: 根据采样率，通道数等配置DAI硬件
- set_sysclk: 设置sysclk
- set_fmt: 配置数据格式
- trigger: 控制音频流的启动和停止, 暂停和恢复等

##### PCM

在原先，DMAengine还不够完善，一般是需要自己编写PCM的部分,这里的PCM不是指硬件PCM（Pulse-Code Modulation），而是说linux里面的数据结构如何传输。

或者有特殊需求的话，也可以自己编写。

在k1的ASoC驱动中，PCM的部分是自己实现的，因为涉及到了Capture无法单独启动和hdmi音频的问题。

后面经过调查，Capture是可以单独启动，并且hdmi音频基本能够不依赖自己写的PCM去实现。

所以现代ASoC的PCM一般都是通过`devm_snd_dmaengine_pcm_register`去使用dmaengine提供的api传输，这套api内部会自动获取dts node的dma节点并申请dma通道去自动完成传输流程，省去很多的研发时间。

##### Codec

驱动由厂商提供，只需要开启config和编写dts node即可。不做过多介绍。

##### Card

上面介绍过Machine层就是一个胶水层，它将Platform和Codec连接起来，形成一个完整的声卡设备。

基本是一个通用操作，所以已经有现成的驱动实现了：simple-card.

想要使用该驱动，只需要将config开启，然后参考dt-binding去编写dts node即可。

这里给出一个例子：

```
sound_card: snd-card@1 {
  compatible = "simple-audio-card";
  simple-audio-card,format = "i2s";
  #address-cells = <1>;
  #size-cells = <0>;

	simple-audio-card,dai-link@0 {
		reg = <0>;
		format = "i2s";
		frame-master = <&link0_cpu>;
		bitclock-master = <&link0_cpu>;

		link0_cpu: cpu {
			sound-dai = <&i2s0>;
		};

		link0_codec: codec {
			sound-dai = <&es8326>;
		};
	};
};
```

其中，每一个dai-link就是一个声卡设备，在dai-link中指定通信协议，frame-master和bitclock-master分别指定了音频数据和时钟的来源。Link0_cpu就是指定cpu的通信接口是哪一个。 link0_codec就是指定codec设备是哪一个，形成一个绑定关系。

### I2C scl clock upstream

在upstream的i2c中，scl时钟的频率依赖ILCR寄存器的默认值去实现。

硬件的默认 ILCR 值 (SLV=0x156, FLV=0x5d) 会导致 SCL 频率低于预期。

例如，在默认的 31.5 MHz 输入时钟下，这些默认设置会导致SCL 频率:
 - 在目标频率为 100 kHz 时约为 93 kHz（标准模式
 - 在目标频率为 400 kHz 时约为 338 kHz（快速模式）。

这些频率低于 100 kHz/400 kHz 的标称速度。

为了解决这个问题，不应该依赖默认值或者在设备树中写死ilcr的值。

应该动态获取dts node中的freq去调整ilcr的值。

#### V0 (PLCT内部版本)

该版本直接获取dts node中的freq保存，创建了一个函数去通过freq设置ilcr的值。

经过Yixun Lan的建议，应引入ccf框架。

#### V1

引入ccf框架，将ilcr寄存器注册为scl clock，在i2c驱动中内部托管。

实现round, recalc, set_rate函数。

经过Yaozi的建议，取消goto并且使用devm_add_action_or_reset函数简化代码。

如原先是：

```
ret = register_xxx()

ret = yyy
if (ret)
  goto exit;

exit:
  unregister_xxx()
```


现在可以使用:

```
ret = register_xxx()

devm_add_action_or_reset(dev, unregister_xxx, param);

ret = yyy
if (ret)
  return -EINVAL;
```

#### V2

添加了一些函数检查。

当dts获取不到合法freq时，默认给一个freq而不是报错退出。

### SSPA I2S_BCLK父时钟upstream修复

在upstream clock驱动中，完全按照了spacemit public document去实现clk.

但是公开文档并不是最新的，导致出现偏差。

在SSPA的CLOCK_RESET寄存器中，BIT6:4(FNCLKSEL)是用来选择SSPA的父时钟的。

但是这里有一个特殊点，就是当FNCLKSEL=7时，为I2S_BCLK作为SSPA的父时钟.

此时BIT3必须为1,否则无效。其他情况下BIT3的值无关紧要。

upstream clock中并没有关注到bit3这一点，所以：

#### V1

在set_parent的时候，重新定义了set_parent_sspa这一个ops.

该ops在index为7的时候设置BIT3为1.

该版本并不完美，代码复用效率不够高。

#### V2

在Yaozi的建议下，跟Jin确认了bit3为1时，并不会对其他的父时钟产生影响。

所以引入virtual(dummy) clock去做父时钟。

这样在级联的影响下，bit3和fcnsel都会选择到正确的值。

代码差异如下：

```
-static const struct clk_parent_data sspa_parents[] = {
+CCU_GATE_DEFINE(sspa0_i2s_bclk, CCU_PARENT_HW(i2s_bclk), APBC_SSPA0_CLK_RST, BIT(3), 0);
+CCU_GATE_DEFINE(sspa1_i2s_bclk, CCU_PARENT_HW(i2s_bclk), APBC_SSPA1_CLK_RST, BIT(3), 0);
+
+static const struct clk_parent_data sspa0_parents[] = {
 	CCU_PARENT_HW(pll1_d384_6p4),
 	CCU_PARENT_HW(pll1_d192_12p8),
 	CCU_PARENT_HW(pll1_d96_25p6),
@@ -357,10 +360,22 @@ static const struct clk_parent_data sspa_parents[] = {
 	CCU_PARENT_HW(pll1_d768_3p2),
 	CCU_PARENT_HW(pll1_d1536_1p6),
 	CCU_PARENT_HW(pll1_d3072_0p8),
-	CCU_PARENT_HW(i2s_bclk),
+	CCU_PARENT_HW(sspa0_i2s_bclk),
 };
-CCU_MUX_GATE_DEFINE(sspa0_clk, sspa_parents, APBC_SSPA0_CLK_RST, 4, 3, BIT(1), 0);
-CCU_MUX_GATE_DEFINE(sspa1_clk, sspa_parents, APBC_SSPA1_CLK_RST, 4, 3, BIT(1), 0);
+CCU_MUX_GATE_DEFINE(sspa0_clk, sspa0_parents, APBC_SSPA0_CLK_RST, 4, 3, BIT(1), 0);
+
+static const struct clk_parent_data sspa1_parents[] = {
+	CCU_PARENT_HW(pll1_d384_6p4),
+	CCU_PARENT_HW(pll1_d192_12p8),
+	CCU_PARENT_HW(pll1_d96_25p6),
+	CCU_PARENT_HW(pll1_d48_51p2),
+	CCU_PARENT_HW(pll1_d768_3p2),
+	CCU_PARENT_HW(pll1_d1536_1p6),
+	CCU_PARENT_HW(pll1_d3072_0p8),
+	CCU_PARENT_HW(sspa1_i2s_bclk),
+};
+CCU_MUX_GATE_DEFINE(sspa1_clk, sspa1_parents, APBC_SSPA1_CLK_RST, 4, 3, BIT(1), 0);
+
 CCU_GATE_DEFINE(dro_clk, CCU_PARENT_HW(apb_clk), APBC_DRO_CLK_RST, BIT(1), 0);
 CCU_GATE_DEFINE(ir_clk, CCU_PARENT_HW(apb_clk), APBC_IR_CLK_RST, BIT(1), 0);
 CCU_GATE_DEFINE(tsen_clk, CCU_PARENT_HW(apb_clk), APBC_TSEN_CLK_RST, BIT(1), 0);
@@ -965,6 +980,8 @@ static struct clk_hw *k1_ccu_apbc_hws[] = {
 	[CLK_SSPA1_BUS]		= &sspa1_bus_clk.common.hw,
 	[CLK_TSEN_BUS]		= &tsen_bus_clk.common.hw,
 	[CLK_IPC_AP2AUD_BUS]	= &ipc_ap2aud_bus_clk.common.hw,
+	[CLK_SSPA0_I2S_BCLK]	= &sspa0_i2s_bclk.common.hw,
+	[CLK_SSPA1_I2S_BCLK]	= &sspa1_i2s_bclk.common.hw,
 };
```

#### V3

在V2中，我在.h文件中添加CLK_SSPAx_I2S_BCLK的定义是直接在中间插入。

经过Yaozi和Rob的提醒，这会导致ABI breaking.

所以在V3中修复了这点。

讨论链接：https://lore.kernel.org/all/20250723044128.GA1207874-robh@kernel.org/

### P1 MTP掉码问题解决

P1寄存器的值来自于内部的MTP模块（eeprom），但由于未知原因，mtp中的数据会出现掉码的问题（码值不对）。

但是现在这批有问题的P1已经贴上了，所以寻找不拆机不飞线的刷写补救方法:

进入刷机模式通过fastboot命令发送bin文件数据并刷写，基础程序是rock提供的baremetal。

#### day 1

- 熟悉pmic读出，擦除，写入流程
- 成功实现读出，写入函数
- 发现无法进行全擦除操作，只可以实现页擦除
- 由于可以读取，擦出，所以没有怀疑是测试模式没进入的原因

#### day 2

- 发现读出数据是中间对称的，查看手册并且与F部确认发现是无法进入测试模式
- 现在无法全擦的原因也找到了，也是因为测试模式没进入成功
- 示波器抓波形发现是发送测试模式seq code的时候被i2c hardware controller自己挂了个stop(进入测试模式的seq中不可以出现stop)
- 随后想到先用0x41(pmic slave addr)作为slave addr,后面的seq code作为数据发送，但是中间必须涉及到restart信号，此时seq又变成slave addr了还是会加入stop.
- 经过讨论研究决定使用sw-i2c去进入test mode

#### day 3
- 开发sw-i2c, 成功进入测试模式，成功全擦除，正常读写，正常写入
- 收到新需求，检测PMIC的版本是否是B,如果是B直接进行刷写
- 修改代码格式，规范，提交到rock维护的baremetal上游

### k3 pinctrl

接到k3 pinctrl驱动编写的任务，首先我先做了如下几件事情：

- 查看k1和k3的pinmux寄存器bit定义，发现完全一致，代码基本可复用
- 查看k1和k3的pinmux base addr，发现完全一致
- 查看k1和k3的引脚定义，发现并不相同，此时需要自己写引脚资料描述了

经过如上调研，我直接拿来了upstream的驱动进行修改，根据Kevin的建议保留pinctrl-k1.c的命名。

并且在Kconfig里面加入K3的支持描述。

在进行引脚资源描述的时候，由于寄存器偏移中间会有中断，所以引脚资源描述并没有根据数据手册上面的comment作为索引，而是保持了顺序, 否则会出现内存资源浪费的情况. 这种情况在k1比较严重，
在k3上面比较简单，仅在130的时候断了8个字节。

由于在引脚资源描述的时候按照了递增顺序而不是手册顺序，所以需要有一个pin_to_offset的函数根据pin计算reg offset值。

为了保持和k1原驱动一致，在匹配data的结构体中增加了指针函数去根据k1和k3指向不同的pin_to_offset.

diff：
```
diff --git a/drivers/pinctrl/spacemit/Kconfig b/drivers/pinctrl/spacemit/Kconfig
index 4aeb8356631e..c740663e8f4f 100644
--- a/drivers/pinctrl/spacemit/Kconfig
+++ b/drivers/pinctrl/spacemit/Kconfig
@@ -4,7 +4,7 @@
 #
 
 config PINCTRL_SPACEMIT_K1
-	bool "SpacemiT K1 SoC Pinctrl driver"
+	bool "SpacemiT K1/K3 SoC Pinctrl driver"
 	depends on SOC_SPACEMIT || COMPILE_TEST
 	depends on OF
 	default SOC_SPACEMIT
@@ -12,7 +12,7 @@ config PINCTRL_SPACEMIT_K1
 	select GENERIC_PINMUX_FUNCTIONS
 	select GENERIC_PINCONF
 	help
-	  Say Y to select the pinctrl driver for K1 SoC.
+	  Say Y to select the pinctrl driver for K1/K3 SoC.
 	  This pin controller allows selecting the mux function for
 	  each pin. This driver can also be built as a module called
 	  pinctrl-k1.
diff --git a/drivers/pinctrl/spacemit/pinctrl-k1.c b/drivers/pinctrl/spacemit/pinctrl-k1.c
index 9996b1c4a07e..b494cf2786de 100644
--- a/drivers/pinctrl/spacemit/pinctrl-k1.c
+++ b/drivers/pinctrl/spacemit/pinctrl-k1.c
@@ -1,5 +1,6 @@
 // SPDX-License-Identifier: GPL-2.0
 /* Copyright (c) 2024 Yixun Lan <dlan@gentoo.org> */
+/* Copyright (c) 2025 Troy Mitchell <troy.mitchell@linux.spacemit.com> */
 
 #include <linux/bits.h>
 #include <linux/clk.h>
@@ -66,6 +67,7 @@ struct spacemit_pinctrl_data {
 	const struct pinctrl_pin_desc   *pins;
 	const struct spacemit_pin	*data;
 	u16				npins;
+	unsigned int			(*pin_to_offset)(unsigned int pin);
 };
 
 struct spacemit_pin_mux_config {
@@ -79,7 +81,7 @@ struct spacemit_pin_drv_strength {
 };
 
 /* map pin id to pinctrl register offset, refer MFPR definition */
-static unsigned int spacemit_pin_to_offset(unsigned int pin)
+static unsigned int spacemit_k1_pin_to_offset(unsigned int pin)
 {
 	unsigned int offset = 0;
 
@@ -124,10 +126,17 @@ static unsigned int spacemit_pin_to_offset(unsigned int pin)
 	return offset << 2;
 }
 
+static unsigned int spacemit_k3_pin_to_offset(unsigned int pin)
+{
+	if (pin > 130)
+		pin += 2;
+	return pin << 2;
+}
+
 static inline void __iomem *spacemit_pin_to_reg(struct spacemit_pinctrl *pctrl,
 						unsigned int pin)
 {
-	return pctrl->regs + spacemit_pin_to_offset(pin);
+	return pctrl->regs + pctrl->data->pin_to_offset(pin);
 }
 
 static u16 spacemit_dt_get_pin(u32 value)
@@ -177,7 +186,7 @@ static void spacemit_pctrl_dbg_show(struct pinctrl_dev *pctldev,
 	void __iomem *reg;
 	u32 value;
 
-	seq_printf(seq, "offset: 0x%04x ", spacemit_pin_to_offset(pin));
+	seq_printf(seq, "offset: 0x%04x ", pctrl->data->pin_to_offset(pin));
 	seq_printf(seq, "type: %s ", io_type_desc[type]);
 
 	reg = spacemit_pin_to_reg(pctrl, pin);
@@ -1042,10 +1051,348 @@ static const struct spacemit_pinctrl_data k1_pinctrl_data = {
 	.pins = k1_pin_desc,
 	.data = k1_pin_data,
 	.npins = ARRAY_SIZE(k1_pin_desc),
+	.pin_to_offset = spacemit_k1_pin_to_offset,
+};
+
+static const struct pinctrl_pin_desc k3_pin_desc[] = {
+	PINCTRL_PIN(0, "GPIO_00"),
...
+	PINCTRL_PIN(152, "PWR_SSP_RXD"),
+};
+
+static const struct spacemit_pin k3_pin_data[ARRAY_SIZE(k3_pin_desc)] = {
+	/* GPIO1 bank */
+	K1_FUNC_PIN(0, 0, IO_TYPE_EXTERNAL),
...
+
+	/* PMIC */
+	K1_FUNC_PIN(128, 0, IO_TYPE_1V8),
+	K1_FUNC_PIN(129, 0, IO_TYPE_1V8),
+	K1_FUNC_PIN(130, 0, IO_TYPE_1V8),
+	K1_FUNC_PIN(131, 0, IO_TYPE_1V8),
+
+	/* SD/MMC1 */
+	K1_FUNC_PIN(132, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(133, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(134, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(135, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(136, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(137, 0, IO_TYPE_EXTERNAL),
+
+	/* QSPI */
+	K1_FUNC_PIN(138, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(139, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(140, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(141, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(142, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(143, 0, IO_TYPE_EXTERNAL),
+	K1_FUNC_PIN(144, 0, IO_TYPE_EXTERNAL),
+
+	/* PMIC */
+	K1_FUNC_PIN(145, 0, IO_TYPE_1V8),
+	K1_FUNC_PIN(146, 0, IO_TYPE_1V8),
+	K1_FUNC_PIN(147, 0, IO_TYPE_1V8),
+	K1_FUNC_PIN(148, 0, IO_TYPE_1V8),
+	K1_FUNC_PIN(149, 0, IO_TYPE_1V8),
+	K1_FUNC_PIN(150, 0, IO_TYPE_1V8),
+	K1_FUNC_PIN(151, 0, IO_TYPE_1V8),
+	K1_FUNC_PIN(152, 0, IO_TYPE_1V8),
+};
+
+static const struct spacemit_pinctrl_data k3_pinctrl_data = {
+	.pins = k3_pin_desc,
+	.data = k3_pin_data,
+	.npins = ARRAY_SIZE(k3_pin_desc),
+	.pin_to_offset = spacemit_k3_pin_to_offset,
 };
 
 static const struct of_device_id k1_pinctrl_ids[] = {
 	{ .compatible = "spacemit,k1-pinctrl", .data = &k1_pinctrl_data },
+	{ .compatible = "spacemit,k3-pinctrl", .data = &k3_pinctrl_data },
 	{ /* sentinel */ }
 };
 MODULE_DEVICE_TABLE(of, k1_pinctrl_ids);
@@ -1061,5 +1408,6 @@ static struct platform_driver k1_pinctrl_driver = {
 builtin_platform_driver(k1_pinctrl_driver);
 
 MODULE_AUTHOR("Yixun Lan <dlan@gentoo.org>");
-MODULE_DESCRIPTION("Pinctrl driver for the SpacemiT K1 SoC");
+MODULE_AUTHOR("Troy Mitchell <troy.mitchell@linux.spacemit.com>");
+MODULE_DESCRIPTION("Pinctrl driver for the SpacemiT Kx SoC");
 MODULE_LICENSE("GPL");
diff --git a/drivers/pinctrl/spacemit/pinctrl-k1.h b/drivers/pinctrl/spacemit/pinctrl-k1.h
index aa2fcd409223..16143fea469e 100644
--- a/drivers/pinctrl/spacemit/pinctrl-k1.h
+++ b/drivers/pinctrl/spacemit/pinctrl-k1.h
@@ -38,4 +38,3 @@ enum spacemit_pin_io_type {
 	}
 
 #endif /* _PINCTRL_SPACEMIT_K1_H */
-
```

### ebpf学习

eBPF（Extended Berkeley Packet Filter）是一个Linux内核中的强大技术，它允许用户空间程序在不修改内核源代码或加载内核模块的情况下，在内核中运行特权程序。

比如可以在不修改内核的情况下为内核增加新的功能。

比如可以在任何一个内核代码上附加一个ebpf程序来调试或监控。

对于高性能监控或跟踪场景，ftrace不够高效，可能会出现卡顿，比如双DIE互联场景，此时使用ebpf来进行性能监控和测试更佳合适。

对于安全，网络性能的监控，ebpf更加合适。

我学习了如下：

- 如何使用python bcc创建ebpf程序
- 如何使用libbpf创建ebpf程序
- awesome ebpf

### JATG & gdb使用

如题。


