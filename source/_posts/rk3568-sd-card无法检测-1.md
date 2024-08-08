title: rk3568 sd card无法检测
date: '2024-07-31 15:00:04'
updated: '2024-07-31 15:00:06'
tags:
  - kernel
  - linux
  - rk3568
  - rockchip
categories:
  - kernel
  - rockchip
---
配置好设备树节点后插入sd卡无法检测。

设备树节点如下：

```
&sdmmc0 {
	bus-width = <4>;
	cap-sd-highspeed;
	cd-gpios = <&gpio0 RK_PA4 GPIO_ACTIVE_LOW>;
	disable-wp;
	pinctrl-names = "default";
	pinctrl-0 = <&sdmmc0_bus4 &sdmmc0_clk &sdmmc0_cmd &sdmmc0_det>;
	sd-uhs-sdr104;
	vmmc-supply = <&vcc3v3_sd>;
	vqmmc-supply = <&vccio_sd>;
	status = "okay";
};
```

查看内核日志发现如下报错：

```
dmesg | grep mmc
[   12.226496] dwmmc_rockchip fe2b0000.mmc: Looking up vmmc-supply from device tree
[   12.226535] dwmmc_rockchip fe2b0000.mmc: Looking up vqmmc-supply from device tree
```

`vmmc-supply`和`vqmmc-supply`由`rk809`提供，`rk809`在该板子上连接到了`i2c0`, 设备树如下：

```
&i2c0 {
	status = "okay";

	rk809: pmic@20 {
		compatible = "rockchip,rk809";
		reg = <0x20>;
		interrupt-parent = <&gpio0>;
		interrupts = <RK_PA3 IRQ_TYPE_LEVEL_LOW>;
		assigned-clocks = <&cru I2S1_MCLKOUT_TX>;
		assigned-clock-parents = <&cru CLK_I2S1_8CH_TX>;
		#clock-cells = <1>;
		clock-names = "mclk";
		clocks = <&cru I2S1_MCLKOUT_TX>;
		pinctrl-names = "default";
		pinctrl-0 = <&pmic_int>;
		rockchip,system-power-controller;
		#sound-dai-cells = <0>;
		vcc1-supply = <&vcc3v3_sys>;
		vcc2-supply = <&vcc3v3_sys>;
		vcc3-supply = <&vcc3v3_sys>;
		vcc4-supply = <&vcc3v3_sys>;
		vcc5-supply = <&vcc3v3_sys>;
		vcc6-supply = <&vcc3v3_sys>;
		vcc7-supply = <&vcc3v3_sys>;
		vcc8-supply = <&vcc3v3_sys>;
		vcc9-supply = <&vcc3v3_sys>;
		wakeup-source;

		regulators {
			vdd_logic: DCDC_REG1 {
				regulator-name = "vdd_logic";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <500000>;
				regulator-max-microvolt = <1350000>;
				regulator-ramp-delay = <6001>;
				regulator-initial-mode = <0x2>;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vdd_gpu: DCDC_REG2 {
				regulator-name = "vdd_gpu";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <500000>;
				regulator-max-microvolt = <1350000>;
				regulator-ramp-delay = <6001>;
				regulator-initial-mode = <0x2>;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vcc_ddr: DCDC_REG3 {
				regulator-name = "vcc_ddr";
				regulator-always-on;
				regulator-boot-on;
				regulator-initial-mode = <0x2>;

				regulator-state-mem {
					regulator-on-in-suspend;
				};
			};

			vdd_npu: DCDC_REG4 {
				regulator-name = "vdd_npu";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <500000>;
				regulator-max-microvolt = <1350000>;
				regulator-ramp-delay = <6001>;
				regulator-initial-mode = <0x2>;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vcc_1v8: DCDC_REG5 {
				regulator-name = "vcc_1v8";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <1800000>;
				regulator-max-microvolt = <1800000>;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vdda0v9_image: LDO_REG1 {
				regulator-name = "vdda0v9_image";
				regulator-boot-on;
				regulator-always-on;
				regulator-min-microvolt = <900000>;
				regulator-max-microvolt = <900000>;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vdda_0v9: LDO_REG2 {
				regulator-name = "vdda_0v9";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <900000>;
				regulator-max-microvolt = <900000>;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vdda0v9_pmu: LDO_REG3 {
				regulator-name = "vdda0v9_pmu";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <900000>;
				regulator-max-microvolt = <900000>;

				regulator-state-mem {
					regulator-on-in-suspend;
					regulator-suspend-microvolt = <900000>;
				};
			};

			vccio_acodec: LDO_REG4 {
				regulator-name = "vccio_acodec";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <3300000>;
				regulator-max-microvolt = <3300000>;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vccio_sd: LDO_REG5 {
				regulator-name = "vccio_sd";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <1800000>;
				regulator-max-microvolt = <3300000>;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vcc3v3_pmu: LDO_REG6 {
				regulator-name = "vcc3v3_pmu";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <3300000>;
				regulator-max-microvolt = <3300000>;

				regulator-state-mem {
					regulator-on-in-suspend;
					regulator-suspend-microvolt = <3300000>;
				};
			};

			vcca_1v8: LDO_REG7 {
				regulator-name = "vcca_1v8";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <1800000>;
				regulator-max-microvolt = <1800000>;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vcca1v8_pmu: LDO_REG8 {
				regulator-name = "vcca1v8_pmu";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <1800000>;
				regulator-max-microvolt = <1800000>;

				regulator-state-mem {
					regulator-on-in-suspend;
					regulator-suspend-microvolt = <1800000>;
				};
			};

			vcca1v8_image: LDO_REG9 {
				regulator-name = "vcca1v8_image";
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <1800000>;
				regulator-max-microvolt = <1800000>;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vcc_3v3: SWITCH_REG1 {
				regulator-name = "vcc_3v3";
				regulator-always-on;
				regulator-boot-on;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};

			vcc3v3_sd: SWITCH_REG2 {
				regulator-name = "vcc3v3_sd";
				regulator-always-on;
				regulator-boot-on;

				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};
		};
	};
};
```

查找`rk809`对应的`compatible`c文件：

```bash
❯ find -name "*.c" -exec grep -nR "rockchip,rk809" {} +

./drivers/mfd/rk8xx-i2c.c:163:	{ .compatible = "rockchip,rk809", .data = &rk809_data },
```

根据该目录下的`Makefile`找到对应配置选项：

```makefile
  1 obj-$(CONFIG_MFD_RK8XX)         += rk8xx-core.o
225 obj-$(CONFIG_MFD_RK8XX_I2C)     += rk8xx-i2c.o
```

`menuconfig`开启以下选项：

```
CONFIG_MFD_RK8XX
CONFIG_MFD_RK8XX_I2C
```

重新编译内核发现还是无法检测到，经过查询发现`RK809`还有其他选项可以打开：

```
CONFIG_RTC_RK808
CONFIG_PINCTRL_RK805
CONFIG_REGULATOR_RK808
CONFIG_INPUT_RK805_PWRKEY
CONFIG_COMMON_CLK_RK808
```

经过启用`CONFIG_REGULATOR_RK808`选项后，可以正常检测并挂载。

没有测试`CONFIG_REGULATOR_RK808`是否依赖于其他几个选项，直接全部开启了。

ref: https://blog.csdn.net/weixin_49303682/article/details/138390797