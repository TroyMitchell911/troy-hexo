title: OpenHarmony on rk3568驱动intel 7260无线网卡
date: '2024-08-21 14:14:14'
updated: '2024-08-21 14:14:18'
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

使用ifconfig能够查看到，但是操作失败：

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

查看有关`rfkill`的内核日志：

```bash
# dmesg | grep rfkill
[    1.441841] [BT_RFKILL]: Enter rfkill_rk_init
[    1.441854] [WLAN_RFKILL]: Enter rfkill_wlan_init
[    1.497371] [WLAN_RFKILL]: rockchip_wifi_get_oob_irq: rfkill-wlan driver has not Successful initialized
[    1.497449] [WLAN_RFKILL]: rockchip_wifi_power: rfkill-wlan driver has not Successful initialized
[    3.909280] [WLAN_RFKILL]: rockchip_wifi_power: rfkill-wlan driver has not Successful initialized
```

## Ref

https://wireless.wiki.kernel.org/en/users/Drivers/iwlwifi
https://www.kernel.org/doc/html/v4.14/driver-api/firmware/built-in-fw.html
https://wiki.gentoo.org/wiki/Linux_firmware