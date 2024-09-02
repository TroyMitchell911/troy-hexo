title: OpenHarmony on rk3568驱动intel 7260无线网卡
date: '2024-08-21 14:14:14'
updated: '2024-09-02 16:03:38'
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

使用以下命令查找`intel 7260`的驱动配置:

```bash
❯ find -name "Kconfig" -exec grep -n "7260" {} +
./drivers/watchdog/Kconfig:689:	  Technologic Systems TS-7200, TS-7250 and TS-7260 boards have
./drivers/net/wireless/intel/iwlwifi/Kconfig:22:		Intel 7260 Wi-Fi Adapter
```

在`./drivers/net/wireless/intel/iwlwifi/Kconfig`找到了其配置，进去查看：

```bash
config IWLWIFI
	tristate "Intel Wireless WiFi Next Gen AGN - Wireless-N/Advanced-N/Ultimate-N (iwlwifi) "
	depends on PCI && HAS_IOMEM && CFG80211
	select FW_LOADER
	help
	  Select to build the driver supporting the:

	  Intel Wireless WiFi Link Next-Gen AGN

	  This option enables support for use with the following hardware:
		Intel Wireless WiFi Link 6250AGN Adapter
		Intel 6000 Series Wi-Fi Adapters (6200AGN and 6300AGN)
		Intel WiFi Link 1000BGN
		Intel Wireless WiFi 5150AGN
		Intel Wireless WiFi 5100AGN, 5300AGN, and 5350AGN
		Intel 6005 Series Wi-Fi Adapters
		Intel 6030 Series Wi-Fi Adapters
		Intel Wireless WiFi Link 6150BGN 2 Adapter
		Intel 100 Series Wi-Fi Adapters (100BGN and 130BGN)
		Intel 2000 Series Wi-Fi Adapters
		Intel 7260 Wi-Fi Adapter
		Intel 3160 Wi-Fi Adapter
		Intel 7265 Wi-Fi Adapter
		Intel 8260 Wi-Fi Adapter
		Intel 3165 Wi-Fi Adapter


	  This driver uses the kernel's mac80211 subsystem.

	  In order to use this driver, you will need a firmware
	  image for it. You can obtain the microcode from:

	          <https://wireless.wiki.kernel.org/en/users/Drivers/iwlwifi>.

	  The firmware is typically installed in /lib/firmware. You can
	  look in the hotplug script /etc/hotplug/firmware.agent to
	  determine which directory FIRMWARE_DIR is set to when the script
	  runs.

	  If you want to compile the driver as a module ( = code which can be
	  inserted in and removed from the running kernel whenever you want),
	  say M here and read <file:Documentation/kbuild/modules.rst>.  The
	  module will be called iwlwifi.

if IWLWIFI

config IWLWIFI_LEDS
	bool
	depends on LEDS_CLASS=y || LEDS_CLASS=IWLWIFI
	depends on IWLMVM || IWLDVM
	select LEDS_TRIGGERS
	select MAC80211_LEDS
	default y

config IWLDVM
	tristate "Intel Wireless WiFi DVM Firmware support"
	depends on MAC80211
	help
	  This is the driver that supports the DVM firmware. The list
	  of the devices that use this firmware is available here:
	  https://wireless.wiki.kernel.org/en/users/drivers/iwlwifi#firmware

config IWLMVM
	tristate "Intel Wireless WiFi MVM Firmware support"
	select WANT_DEV_COREDUMP
	depends on MAC80211
	help
	  This is the driver that supports the MVM firmware. The list
	  of the devices that use this firmware is available here:
	  https://wireless.wiki.kernel.org/en/users/drivers/iwlwifi#firmware
```

创建文件夹并拷贝`firmware`：

```bash
mkdir -p device/board/hihope/rk3568/kernel/firmware/ && cp <firmware_path> device/board/hihope/rk3568/kernel/firmware/
```

修改编译脚本：

```bash
❯ diff device/board/hihope/rk3568/kernel/build_kernel.sh device/board/hihope/rk3568/kernel/build_kernel.sh.1
77,81d76
< # firmware
< mkdir -p ${KERNEL_SRC_TMP_PATH}/kernel/firmware
< cp  -rf ${3}/kernel/firmware/* ${KERNEL_SRC_TMP_PATH}/kernel/firmware
< 
< 
```

查看日志初始化成功：

```bash
# dmesg | grep iwl
[    2.899279] iwlwifi 0000:01:00.0: enabling device (0000 -> 0002)
[    2.902993] iwlwifi 0000:01:00.0: loaded firmware version 17.3216344376.0 7260-17.ucode op_mode iwlmvm
[    2.903274] iwlwifi 0000:01:00.0: Detected Intel(R) Dual Band Wireless AC 7260, REV=0x144
[    2.909365] iwlwifi 0000:01:00.0: reporting RF_KILL (radio disabled)
[    2.909427] iwlwifi 0000:01:00.0: RF_KILL bit toggled to disable radio.
[    2.950167] iwlwifi 0000:01:00.0: base HW address: 80:86:f2:c3:ff:00
[    2.969690] ieee80211 phy0: Selected rate control algorithm 'iwl-mvm-rs'
```

使用`ifconfig`能够查看到，但是操作失败：

```bash
# ifconfig wlan0
wlan0     Link encap:Ethernet  HWaddr 80:86:f2:c3:ff:00  Driver iwlwifi
          BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0 
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0 
          collisions:0 txqueuelen:1000 
          RX bytes:0 TX bytes:0 

# ifconfig wlan0 up
ifconfig: ioctl 8914: No error information
```

使用rfkill查看状态：

```bash
# rfkill list
0: phy0: wlan
        Soft blocked: no
        Hard blocked: yes
```

发现被锁定了，尝试解锁但是失败了..：

```bash
# rfkill unblock all
# rfkill list                                                                  
0: phy0: wlan
        Soft blocked: no
        Hard blocked: yes
```

根据原理图和设备树，我们锁定两个引脚，一个是`reset`一个是`disable`:

```c
/* mini pcie */
&pcie2x1 {
	reset-gpios = <&gpio3 RK_PC1 GPIO_ACTIVE_HIGH>;
	disable-gpios = <&gpio3 RK_PC2 GPIO_ACTIVE_HIGH>;
	vpcie3v3-supply = <&mini_pcie_3v3>;
	status = "okay";
};
```

![2024-09-02-15-56-00.png](https://i.postimg.cc/Y2TpH1Pq/2024-09-02-15-56-00.png)

经过万用表测量`reset`引脚在进入`shell`后是高电平，而`disable`引脚则从上电到进入`shell`一直都是低电平。

那么应该可以确定问题所在了，就是`disable`搞得鬼。

查看文档`Documentation/devicetree/bindings/pci/rockchip-pcie-host.txt`中并没有关于`disable`的说明，这让我大感奇怪：

```bash
❯ cat Documentation/devicetree/bindings/pci/rockchip-pcie-host.txt | grep disable
```

因为这个设备树的基准文件来自于野火`sdk`的`4.19`版本，所以这时候还在怀疑是不是`5.10`内核的驱动中取消了这一个节点属性换成其他名字的了，于是又去`4.19`的文档看了一下，也是没有`disable`的说明，此时心中已经警惕了，感觉就是野火改了驱动。

查找该驱动的c文件：

```bash
❯ find -name "*.c" -exec grep -n "rockchip,rk3568-pcie" {} +
./drivers/phy/rockchip/phy-rockchip-snps-pcie3.c:262:	{ .compatible = "rockchip,rk3568-pcie3-phy", .data = &rk3568_ops },
./drivers/pci/controller/dwc/pcie-dw-rockchip.c:1324:		.compatible = "rockchip,rk3568-pcie",
./drivers/pci/controller/dwc/pcie-dw-rockchip.c:1328:		.compatible = "rockchip,rk3568-pcie-ep",
```

锁定`./drivers/pci/controller/dwc/pcie-dw-rockchip.c`文件，进去一看，果然...:

```c
static int rk_pcie_resource_get(struct platform_device *pdev,
					 struct rk_pcie *rk_pcie)
{
	...
	rk_pcie->dis_gpio = devm_gpiod_get_optional(&pdev->dev, "disable",
						    GPIOD_OUT_LOW);
	if (IS_ERR(rk_pcie->dis_gpio)) {
		dev_err(&pdev->dev, "invalid disable-gpios property in node\n");
		return PTR_ERR(rk_pcie->rst_gpio);
	}
	...
}
```

上述代码块中的内容是野火增加的`disable`引脚资源获取，在官方的驱动里面是并没有这一部分的，所以增加如下内容：

```c
diff --git a/drivers/pci/controller/dwc/pcie-dw-rockchip.c b/drivers/pci/controller/dwc/pcie-dw-rockchip.c
index fa40f51..3c6cd4f 100755
--- a/drivers/pci/controller/dwc/pcie-dw-rockchip.c
+++ b/drivers/pci/controller/dwc/pcie-dw-rockchip.c
@@ -133,6 +133,7 @@ struct rk_pcie {
 	unsigned int			clk_cnt;
 	struct reset_bulk_data		*rsts;
 	struct gpio_desc		*rst_gpio;
+	struct gpio_desc		*dis_gpio;
 	phys_addr_t			mem_start;
 	size_t				mem_size;
 	struct pcie_port		pp;
@@ -892,6 +893,11 @@ static int rk_pcie_host_init(struct pcie_port *pp)
 {
 	int ret;
 	struct dw_pcie *pci = to_dw_pcie_from_pp(pp);
+	struct rk_pcie *rk_pcie = to_rk_pcie(pci);
+        
+	if (!IS_ERR(rk_pcie->dis_gpio)) {
+		gpiod_set_value_cansleep(rk_pcie->dis_gpio, 1);
+        }
 
 	dw_pcie_setup_rc(pp);
 
@@ -1066,6 +1072,13 @@ static int rk_pcie_resource_get(struct platform_device *pdev,
 		dev_err(&pdev->dev, "invalid reset-gpios property in node\n");
 		return PTR_ERR(rk_pcie->rst_gpio);
 	}
+	
+	rk_pcie->dis_gpio = devm_gpiod_get_optional(&pdev->dev, "disable",
+						    GPIOD_OUT_LOW);
+	if (IS_ERR(rk_pcie->dis_gpio)) {
+		dev_err(&pdev->dev, "invalid disable-gpios property in node\n");
+		return PTR_ERR(rk_pcie->rst_gpio);
+	}
 
 	return 0;
 }
-- 
2.34.1
```

重新编译烧录内核，一切正常：

```bash
# rfkill list                                                                  
0: hci0: bluetooth
        Soft blocked: no
        Hard blocked: no
1: phy0: wlan
        Soft blocked: no
        Hard blocked: no
```


## Ref

https://wireless.wiki.kernel.org/en/users/Drivers/iwlwifi
https://www.kernel.org/doc/html/v4.14/driver-api/firmware/built-in-fw.html
https://wiki.gentoo.org/wiki/