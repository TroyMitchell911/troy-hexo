title: '[全志A33-Vstar]Kernel'
date: '2024-09-20 22:49:02'
updated: '2024-09-21 00:15:42'
tags:
  - a33
  - kernel
  - allwinner
---
**本文章基于该uboot启动**：[传送门](https://blog.troy-y.org/2024/09/13/%E5%85%A8%E5%BF%97A33-Uboot/)

## Env

cpu: allwinner a33
board: vstar
host: ubuntu 22.04

需要安装交叉编译工具链：

```bash
❯ sudo apt install gcc-arm-none-eabi
```

## mainline

首先下载主线kernel的源码：

```bash
❯ git clone git@github.com:torvalds/linux.git
```

配置文件的话，在`arch/arm/configs`目录下只有一个`sunxi_defconfig`是我们能够使用的，那么就使用这个配置文件作为基准。

设备树就选择sinlinx的，与vstar开发板相近。

与uboot套路类似，先拷贝一份配置文件和设备树作为vstar开发板的：

```bash
❯ cp arch/arm/configs/sunxi_defconfig arch/arm/configs/a33_vstar_defconfig
❯ cp arch/arm/boot/dts/allwinner/sun8i-a33-sinlinx-sina33.dts arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts
```

修改Makefile支持该设备树：

```bash
diff --git a/arch/arm/boot/dts/allwinner/Makefile b/arch/arm/boot/dts/allwinner/Makefile
index cd0d044882cf..d548f4a2621a 100644
--- a/arch/arm/boot/dts/allwinner/Makefile
+++ b/arch/arm/boot/dts/allwinner/Makefile
@@ -215,6 +215,7 @@ dtb-$(CONFIG_MACH_SUN8I) += \
        sun8i-a33-olinuxino.dtb \
        sun8i-a33-q8-tablet.dtb \
        sun8i-a33-sinlinx-sina33.dtb \
+       sun8i-a33-vstar.dtb \
        sun8i-a83t-allwinner-h8homlet-v2.dtb \
        sun8i-a83t-bananapi-m3.dtb \
        sun8i-a83t-cubietruck-plus.dtb \
```

使用刚才拷贝的配置文件进行编译：

```bash
❯ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- a33_vstar_defconfig
❯ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16
```

可以看到镜像文件和设备树都已经生成：

```bash
❯ ls arch/arm/boot/zImage
arch/arm/boot/zImage
❯ ls arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dtb
arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dtb
```

使用[这篇文章](https://blog.troy-y.org/2024/09/20/sd%E4%B8%8Emmc%E5%88%B7%E5%86%99%E6%8C%87%E5%8D%97/)的技巧将kernel和dtb装入sd卡boot分区，装入emmc的boot分区也是可以的。

**PS：其实这里使用tftp更加合理，但在家中没有网口环境，所以只能退而求其次这样调试了。**

在uboot中执行如下命令查看boot分区：

```bash
=> fatls mmc 0:1
    65536   uboot.env
    23345   sun8i-a33-vstar.dtb
  5496008   zImage

3 file(s), 0 dir(s)
```

现在文件已经装好了，就差装载地址了，在uboot中查看kernel的加载地址和dtb的加载地址：

```bash
=> printenv kernel_addr_r 
kernel_addr_r=0x42000000
=> printenv fdt_addr_r 
fdt_addr_r=0x43000000
```

可以确定内核的加载地址是`0x42000000`，设备树的加载地址是`0x43000000`.

使用load命令加载：

```bash
=> load mmc 0:1 0x42000000 zImage
5496008 bytes read in 229 ms (22.9 MiB/s)
=> load mmc 0:1 0x43000000 sun8i-a33-vstar.dtb
23345 bytes read in 3 ms (7.4 MiB/s)
```

由于内核镜像是经过压缩的，所以这里使用`bootz`命令进行启动：

```bash
=> bootz 0x42000000 - 0x43000000
Kernel image @ 0x42000000 [ 0x000000 - 0x53dcc8 ]
## Flattened Device Tree blob at 43000000
   Booting using the fdt blob at 0x43000000
Working FDT set to 43000000
   Loading Device Tree to 49ff7000, end 49fffb30 ... OK
Working FDT set to 49ff7000

Starting kernel ...

cyclic function video_init took too long: 27000us vs 5000us max
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 6.11.0-rc6-00309-gd2742e387fe4 (troy@troy-WUJIE14-PRO) (arm-linux-gnueabihf-gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, GNU ld (GNU Bin
utils for Ubuntu) 2.38) #4 SMP Fri Sep 20 22:58:32 CST 2024
[    0.000000] CPU: ARMv7 Processor [410fc075] revision 5 (ARMv7), cr=10c5387d
[    0.000000] CPU: div instructions available: patching division code
[    0.000000] CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache
[    0.000000] OF: fdt: Machine model: Sinlinx SinA33
[    0.000000] Memory policy: Data cache writealloc
[    0.000000] cma: Reserved 16 MiB at 0x5d000000 on node -1
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000040000000-0x000000005dffffff]
[    0.000000]   HighMem  empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000040000000-0x000000005dffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000040000000-0x000000005dffffff]
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: Using PSCI v0.1 Function IDs from DT
[    0.000000] percpu: Embedded 13 pages/cpu s20876 r8192 d24180 u53248
[    0.000000] Kernel command line: 
[    0.000000] Dentry cache hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.000000] Inode-cache hash table entries: 32768 (order: 5, 131072 bytes, linear)
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 122880
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=4, Nodes=1
[    0.000000] rcu: Hierarchical RCU implementation.
[    0.000000] rcu:     RCU restricting CPUs from NR_CPUS=8 to nr_cpu_ids=4.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 10 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=4
[    0.000000] NR_IRQS: 16, nr_irqs: 16, preallocated irqs: 16
[    0.000000] GIC: Using split EOI/Deactivate mode
[    0.000000] rcu: srcu_init: Setting srcu_struct sizes based on contention.
[    0.000000] arch_timer: cp15 timer(s) running at 24.00MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x588fe9dc0, max_idle_ns: 440795202592 ns
[    0.000002] sched_clock: 56 bits at 24MHz, resolution 41ns, wraps every 4398046511097ns
[    0.000018] Switching to timer-based delay loop, resolution 41ns
[    0.000214] clocksource: timer: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 79635851949 ns
[    0.000716] Console: colour dummy device 80x30
[    0.000737] printk: legacy console [tty0] enabled
[    0.001150] Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS (lpj=240000)
[    0.001186] CPU: Testing write buffer coherency: ok
[    0.001256] pid_max: default: 32768 minimum: 301
[    0.001462] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.001501] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.002532] /cpus/cpu@0 missing clock-frequency property
[    0.002602] /cpus/cpu@1 missing clock-frequency property
[    0.002636] /cpus/cpu@2 missing clock-frequency property
[    0.002669] /cpus/cpu@3 missing clock-frequency property
[    0.002694] CPU0: thread -1, cpu 0, socket 0, mpidr 80000000
[    0.003911] Setting up static identity map for 0x40100000 - 0x40100060
[    0.004138] rcu: Hierarchical SRCU implementation.
[    0.004168] rcu:     Max phase no-delay instances is 1000.
[    0.004475] Timer migration: 1 hierarchy levels; 8 children per group; 1 crossnode level
[    0.005197] smp: Bringing up secondary CPUs ...
[    0.006381] CPU1: thread -1, cpu 1, socket 0, mpidr 80000001
[    0.007615] CPU2: thread -1, cpu 2, socket 0, mpidr 80000002
[    0.008778] CPU3: thread -1, cpu 3, socket 0, mpidr 80000003
[    0.008903] smp: Brought up 1 node, 4 CPUs
[    0.008972] SMP: Total of 4 processors activated (192.00 BogoMIPS).
[    0.008993] CPU: All CPU(s) started in HYP mode.
[    0.009008] CPU: Virtualization extensions available.
[    0.009739] Memory: 455244K/491520K available (8192K kernel code, 954K rwdata, 2404K rodata, 1024K init, 261K bss, 18116K reserved, 16384K cma-reserved, 0K high
mem)
[    0.010473] devtmpfs: initialized
[    0.016216] VFP support v0.3: implementor 41 architecture 2 part 30 variant 7 rev 5
[    0.016523] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.016574] futex hash table entries: 1024 (order: 4, 65536 bytes, linear)
[    0.017387] pinctrl core: initialized pinctrl subsystem
[    0.019573] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.020760] DMA: preallocated 256 KiB pool for atomic coherent allocations
[    0.021844] thermal_sys: Registered thermal governor 'step_wise'
[    0.022011] hw-breakpoint: found 5 (+1 reserved) breakpoint and 4 watchpoint registers.
[    0.022063] hw-breakpoint: maximum watchpoint size is 8 bytes.
[    0.026636] platform 1c0c000.lcd-controller: Fixed dependency cycle(s) with /soc/drc@1e70000
[    0.029751] platform 1e00000.display-frontend: Fixed dependency cycle(s) with /soc/display-backend@1e60000
[    0.030124] platform 1e00000.display-frontend: Fixed dependency cycle(s) with /soc/display-backend@1e60000
[    0.030236] platform 1e60000.display-backend: Fixed dependency cycle(s) with /soc/drc@1e70000
[    0.030272] platform 1e60000.display-backend: Fixed dependency cycle(s) with /soc/display-frontend@1e00000
[    0.030575] platform 1e60000.display-backend: Fixed dependency cycle(s) with /soc/drc@1e70000
[    0.030674] platform 1c0c000.lcd-controller: Fixed dependency cycle(s) with /soc/drc@1e70000
[    0.030773] platform 1e70000.drc: Fixed dependency cycle(s) with /soc/lcd-controller@1c0c000
[    0.030874] platform 1e70000.drc: Fixed dependency cycle(s) with /soc/display-backend@1e60000
[    0.033511] platform 1c0c000.lcd-controller: Fixed dependency cycle(s) with /panel
[    0.033627] platform panel: Fixed dependency cycle(s) with /soc/lcd-controller@1c0c000
[    0.037105] SCSI subsystem initialized
[    0.037662] usbcore: registered new interface driver usbfs
[    0.037724] usbcore: registered new interface driver hub
[    0.037801] usbcore: registered new device driver usb
[    0.038063] mc: Linux media interface: v0.10
[    0.038129] videodev: Linux video capture interface: v2.00
[    0.038229] pps_core: LinuxPPS API ver. 1 registered
[    0.038250] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.038287] PTP clock support registered
[    0.038843] Advanced Linux Sound Architecture Driver Initialized.
[    0.040004] clocksource: Switched to clocksource arch_sys_counter
[    0.049575] NET: Registered PF_INET protocol family
[    0.049898] IP idents hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.051275] tcp_listen_portaddr_hash hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.051335] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.051366] TCP established hash table entries: 4096 (order: 2, 16384 bytes, linear)
[    0.051427] TCP bind hash table entries: 4096 (order: 4, 65536 bytes, linear)
[    0.051643] TCP: Hash tables configured (established 4096 bind 4096)
[    0.051791] UDP hash table entries: 256 (order: 1, 8192 bytes, linear)
[    0.051856] UDP-Lite hash table entries: 256 (order: 1, 8192 bytes, linear)
[    0.052082] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.052808] RPC: Registered named UNIX socket transport module.
[    0.052855] RPC: Registered udp transport module.
[    0.052873] RPC: Registered tcp transport module.
[    0.052888] RPC: Registered tcp-with-tls transport module.
[    0.052904] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.054149] workingset: timestamp_bits=30 max_order=17 bucket_order=0
[    0.055164] NFS: Registering the id_resolver key type
[    0.055249] Key type id_resolver registered
[    0.055269] Key type id_legacy registered
[    0.055406] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 246)
[    0.055436] io scheduler mq-deadline registered
[    0.055455] io scheduler kyber registered
[    0.055487] io scheduler bfq registered
[    0.123068] Serial: 8250/16550 driver, 8 ports, IRQ sharing disabled
[    0.132571] panel-simple panel: Specify missing connector_type
[    0.140190] lima 1c40000.gpu: gp - mali400 version major 1 minor 1
[    0.140335] lima 1c40000.gpu: pp0 - mali400 version major 1 minor 1
[    0.140452] lima 1c40000.gpu: pp1 - mali400 version major 1 minor 1
[    0.140513] lima 1c40000.gpu: l2_cache0 64K, 4-way, 64byte cache line, 64bit external bus
[    0.140619] lima 1c40000.gpu: bus rate = 200000000
[    0.140646] lima 1c40000.gpu: mod rate = 384000000
[    0.140761] lima 1c40000.gpu: error -ENODEV: _opp_set_regulators: no regulator (mali) found
[    0.141326] lima 1c40000.gpu: Failed to register cooling device
[    0.141920] [drm] Initialized lima 1.1.0 for 1c40000.gpu on minor 0
[    0.146975] CAN device driver interface
[    0.150247] sun6i-rtc 1f00000.rtc: registered as rtc0
[    0.150323] sun6i-rtc 1f00000.rtc: setting system clock to 1970-01-01T00:43:53 UTC (2633)
[    0.150930] i2c_dev: i2c /dev entries driver
[    0.153101] sunxi-wdt 1c20ca0.watchdog: Watchdog enabled (timeout=16 sec, nowayout=0)
[    0.154435] sun4i-ss 1c15000.crypto-engine: Die ID 5
[    0.155623] usbcore: registered new interface driver usbhid
[    0.155654] usbhid: USB HID core driver
[    0.161519] cedrus 1c0e000.video-codec: Device registered as /dev/video0
[    0.165699] NET: Registered PF_PACKET protocol family
[    0.165755] can: controller area network core
[    0.165827] NET: Registered PF_CAN protocol family
[    0.165851] can: raw protocol
[    0.165869] can: broadcast manager protocol
[    0.165891] can: netlink gateway - max_hops=1
[    0.166091] Key type dns_resolver registered
[    0.166263] Registering SWP/SWPB emulation handler
[    0.188378] gpio gpiochip0: Static allocation of GPIO base is deprecated, use dynamic allocation.
[    0.190314] sun8i-a23-r-pinctrl 1f02c00.pinctrl: initialized sunXi PIO driver
[    0.191090] gpio gpiochip1: Static allocation of GPIO base is deprecated, use dynamic allocation.
[    0.193964] sun8i-a33-pinctrl 1c20800.pinctrl: initialized sunXi PIO driver
[    0.194633] sun8i-a33-pinctrl 1c20800.pinctrl: supply vcc-pb not found, using dummy regulator
[    0.216310] 1c28000.serial: ttyS0 at MMIO 0x1c28000 (irq = 140, base_baud = 1500000) is a U6_16550A
[    0.216423] printk: legacy console [ttyS0] enabled
[    1.162066] sun8i-a33-pinctrl 1c20800.pinctrl: supply vcc-pd not found, using dummy regulator
[    1.171186] sun4i-drm display-engine: bound 1e00000.display-frontend (ops 0xc095450c)
[    1.179400] sun4i-drm display-engine: bound 1e60000.display-backend (ops 0xc0953c78)
[    1.187285] sun4i-drm display-engine: bound 1e70000.drc (ops 0xc09537a8)
[    1.194715] sun4i-drm display-engine: bound 1c0c000.lcd-controller (ops 0xc0952418)
[    1.203810] [drm] Initialized sun4i-drm 1.0.0 for display-engine on minor 1
[    1.212105] usb_phy_generic usb_phy_generic.1.auto: dummy supplies not allowed for exclusive requests (id=vbus)
[    1.225635] ehci-platform 1c1a000.usb: EHCI Host Controller
[    1.231359] ehci-platform 1c1a000.usb: new USB bus registered, assigned bus number 1
[    1.232634] sun8i-a23-r-pinctrl 1f02c00.pinctrl: supply vcc-pl not found, using dummy regulator
[    1.232655] ohci-platform 1c1a400.usb: Generic Platform OHCI controller
[    1.232680] ohci-platform 1c1a400.usb: new USB bus registered, assigned bus number 2
[    1.262744] sunxi-rsb 1f03400.rsb: RSB running at 3000000 Hz
[    1.268854] axp20x-rsb sunxi-rsb-3a3: AXP20x variant AXP223 found
[    1.269008] ohci-platform 1c1a400.usb: irq 144, io mem 0x01c1a400
[    1.278800] input: axp20x-pek as /devices/platform/soc/1f03400.rsb/sunxi-rsb-3a3/axp221-pek/input/input0
[    1.291724] axp20x-adc axp22x-adc: DMA mask not set
[    1.297302] ehci-platform 1c1a000.usb: irq 143, io mem 0x01c1a000
[    1.297733] axp20x-battery-power-supply axp20x-battery-power-supply: DMA mask not set
[    1.312539] axp20x-ac-power-supply axp20x-ac-power-supply: DMA mask not set
[    1.324341] axp20x-rsb sunxi-rsb-3a3: AXP20X driver loaded
[    1.330230] ehci-platform 1c1a000.usb: USB 2.0 started, EHCI 1.00
[    1.332049] input: 1c22800.lradc as /devices/platform/soc/1c22800.lradc/input/input1
[    1.337352] hub 1-0:1.0: USB hub found
[    1.348018] hub 1-0:1.0: 1 port detected
[    1.358445] sun8i-a33-pinctrl 1c20800.pinctrl: supply vcc-pf not found, using dummy regulator
[    1.358514] clk: Disabling unused clocks
[    1.358756] hub 2-0:1.0: USB hub found
[    1.358820] hub 2-0:1.0: 1 port detected
[    1.358836] sun8i-a33-pinctrl 1c20800.pinctrl: supply vcc-pc not found, using dummy regulator
[    1.387380] ALSA device list:
[    1.390398]   No soundcards found.
[    1.394999] sunxi-mmc 1c0f000.mmc: Got CD GPIO
[    1.419939] sunxi-mmc 1c11000.mmc: initialized, max. request size: 16384 KB
[    1.423106] sunxi-mmc 1c0f000.mmc: initialized, max. request size: 16384 KB
[    1.434628] /dev/root: Can't open blockdev
[    1.438774] VFS: Cannot open root device "" or unknown-block(0,0): error -6
[    1.445782] Please append a correct "root=" boot option; here are the available partitions:
[    1.454156] List of all bdev filesystems:
[    1.458168]  ext3
[    1.458174]  ext2
[    1.460115]  ext4
[    1.462045]  vfat
[    1.463974] 
[    1.467258] mmc1: host does not support reading read-only switch, assuming write-enable
[    1.467391] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
[    1.467401] CPU: 3 UID: 0 PID: 1 Comm: swapper/0 Not tainted 6.11.0-rc6-00309-gd2742e387fe4 #4
[    1.467411] Hardware name: Allwinner sun8i Family
[    1.467416] Call trace: 
[    1.467430]  unwind_backtrace from show_stack+0x10/0x14
[    1.467465]  show_stack from dump_stack_lvl+0x50/0x64
[    1.467484]  dump_stack_lvl from panic+0x10c/0x344
[    1.467497]  panic from mount_root_generic+0x1d4/0x278
[    1.467513]  mount_root_generic from prepare_namespace+0x1fc/0x258
[    1.467527]  prepare_namespace from kernel_init+0x1c/0x12c
[    1.467542]  kernel_init from ret_from_fork+0x14/0x28
[    1.467554] Exception stack(0xde821fb0 to 0xde821ff8)
[    1.467565] 1fa0:                                     00000000 00000000 00000000 00000000
[    1.467574] 1fc0: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[    1.467582] 1fe0: 00000000 00000000 00000000 00000000 00000013 00000000
[    1.564419] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```

内核直接报了`panic`，经过定位，可以确定如下一条依据：

```bash
[    1.445782] Please append a correct "root=" boot option; here are the available partitions:
```

这里的意思就是我们没有指定根文件系统，那一定是bootargs出了问题，复位进入uboot，查看bootargs的参数：

```bash
=> printenv bootargs
## Error: "bootargs" not defined
```

居然没有这个环境变量，想到了armbian是有对a33进行支持的，我们可以去脱一下armbian的bootargs。

下载[armbian](https://github.com/armbian/community/releases/download/24.11.0-trunk.66/Armbian_community_24.11.0-trunk.66_Lime-a33_bookworm_current_6.6.44_minimal.img.xz)的镜像之后通过`balenaEtcher`工具刷入sd卡，进入系统后提取`boot`分区中的`boot.scr`。

该脚本中有这么一句话：

```bash
setenv bootargs "root=${rootdev} rootwait rootfstype=${rootfstype} ${consoleargs} hdmi.audio=EDID:0 disp.screen0_output_mode=${disp_mode} consoleblank=0 loglevel=${verbosity} ubootpart=${partuuid} ubootsource=${devtype} usb-storage.quirks=${usbstoragequirks} ${extraargs} ${extraboardargs}"
```

在`armbian`的`uboot`中一句句对照env，并且删减一些无关项后得到如下`bootargs`：

```bash
=> setenv bootargs 'earlycon earlyprintk root=/dev/mmcblk0p2 rootwait rw rootfstype=ext4 console=ttyS2,115200 consoleblank=0 loglevel=8'
=> printenv bootargs 
bootargs=earlycon earlyprintk root=/dev/mmcblk0p2 rootwait rw rootfstype=ext4 console=ttyS2,115200 consoleblank=0 loglevel=8
=> saveenv
Saving Environment to FAT... OK
```

重新加载并且启动后发现log到Starting kernel就停止了：

```bash
=> load mmc 0:1 0x42000000 zImage
5496008 bytes read in 229 ms (22.9 MiB/s)
=> load mmc 0:1 0x43000000 sun8i-a33-vstar.dtb
23345 bytes read in 2 ms (11.1 MiB/s)
=> bootz 0x42000000 - 0
Kernel image @ 0x42000000 [ 0x000000 - 0x53dcc8 ]
ERROR: Did not find a cmdline Flattened Device Tree
Could not find a valid device tree
=> bootz 0x42000000 - 0x43000000
Kernel image @ 0x42000000 [ 0x000000 - 0x53dcc8 ]
## Flattened Device Tree blob at 43000000
   Booting using the fdt blob at 0x43000000
Working FDT set to 43000000
   Loading Device Tree to 49ff7000, end 49fffb30 ... OK
Working FDT set to 49ff7000

Starting kernel ...

cyclic function video_init took too long: 27000us vs 5000us max
```

### 修改串口

这很容易判断，我们硬件电气连接的是uart2, 肯定是串口的问题了，查看设备树：

```bash
❯ cat arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts | grep serial
		serial0 = &uart0;
		stdout-path = "serial0:115200n8";
```

果然，设备树中是串口0,为了避免和sd卡引脚复用冲突（**该部分在uboot已经说明过**）, 将其修改成uart2,并且我们的bootargs中指定的也是uart2.

修改后的差异：

```bash
diff --git a/arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts b/arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts
index 0c82ff3c7cb4..7bb29d78c77f 100644
--- a/arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts
+++ b/arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts
@@ -54,11 +54,11 @@ / {
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
@@ -261,6 +261,12 @@ tcon0_out_panel: endpoint@0 {
 &uart0 {
 	pinctrl-names = "default";
 	pinctrl-0 = <&uart0_pb_pins>;
+	status = "disabled";
+};
+
+&uart2 {
+	pinctrl-names = "default";
+	pinctrl-0 = <&uart2_pins>;
 	status = "okay";
 };
 
@@ -273,3 +279,10 @@ &usbphy {
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
```

重新启动后可以正常看到输出了！

```bash
[    1.242405] Waiting for root device /dev/mmcblk0p2...
[    1.278711] mmc1: host does not support reading read-only switch, assuming write-enable
[    1.296810] mmc1: new high speed SDXC card at address 59b4
[    1.307496] mmcblk1: mmc1:59b4 SD64G 59.4 GiB
[    1.326969]  mmcblk1: p1 p2
[    1.381752] mmc0: new DDR MMC card at address 0001
[    1.387723] mmcblk0: mmc0:0001 004G60 3.69 GiB
[    1.405833]  mmcblk0: p1 p2
[    1.414082] mmcblk0boot0: mmc0:0001 004G60 2.00 MiB
[    1.421045] mmcblk0boot1: mmc0:0001 004G60 2.00 MiB
[    1.447615] usb 1-1: new high-speed USB device number 2 using ehci-platform
[    1.462789] EXT4-fs (mmcblk0p2): recovery complete
[    1.467843] EXT4-fs (mmcblk0p2): mounted filesystem 3cdb3fe9-0a84-41be-9519-ec406d029db6 r/w with ordered data mode. Quota mode: disabled.
[    1.480404] VFS: Mounted root (ext4 filesystem) on device 179:10.
[    1.487217] devtmpfs: error mounting -2
[    1.496557] Freeing unused kernel image (initmem) memory: 1024K
[    1.502730] Run /sbin/init as init process
[    1.506831]   with arguments:
[    1.509826]     /sbin/init
[    1.512536]     earlyprintk
[    1.515329]   with environment:
[    1.518489]     HOME=/
[    1.520938]     TERM=linux
[    1.523983] Run /etc/init as init process
[    1.528010]   with arguments:
[    1.531062]     /etc/init
[    1.533975]     earlyprintk
[    1.536766]   with environment:
[    1.539921]     HOME=/
[    1.542366]     TERM=linux
[    1.545366] Run /bin/init as init process
[    1.549385]   with arguments:
[    1.552349]     /bin/init
[    1.554966]     earlyprintk
[    1.557768]   with environment:
[    1.560905]     HOME=/
[    1.563261]     TERM=linux
[    1.565979] Run /bin/sh as init process
[    1.569826]   with arguments:
[    1.572790]     /bin/sh
[    1.575233]     earlyprintk
[    1.578042]   with environment:
[    1.581179]     HOME=/
[    1.583537]     TERM=linux
[    1.586247] Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance.
[    1.600399] CPU: 1 UID: 0 PID: 1 Comm: swapper/0 Not tainted 6.11.0-rc6-00309-gd2742e387fe4-dirty #5
[    1.609523] Hardware name: Allwinner sun8i Family
[    1.614222] Call trace: 
[    1.614238]  unwind_backtrace from show_stack+0x10/0x14
[    1.622013]  show_stack from dump_stack_lvl+0x50/0x64
[    1.627074]  dump_stack_lvl from panic+0x10c/0x344
[    1.631869]  panic from _cpu_down+0x0/0x6a0
[    1.636155] ---[ end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for gu
idance. ]---
```

### 固定mmc序号

但这里还是出现了`pannic`，重点观察到如下内容：

```bash
[    1.242405] Waiting for root device /dev/mmcblk0p2...
[    1.278711] mmc1: host does not support reading read-only switch, assuming write-enable
[    1.296810] mmc1: new high speed SDXC card at address 59b4
[    1.307496] mmcblk1: mmc1:59b4 SD64G 59.4 GiB
```

这里说是等待mmcblk0设备但是sd卡确实mmc1, 并且我每次复位sd卡的序号都不一样，可能是mmc0也可能是mmc1, 这可不行，因为rootfs的分区是我们在bootargs里面定死了的，所以只能固定mmc序号。

修改设备树：

```bash
diff --git a/arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts b/arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts
index f89a005227d9..95a7c7f394b8 100644
--- a/arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts
+++ b/arch/arm/boot/dts/allwinner/sun8i-a33-vstar.dts
@@ -55,6 +55,10 @@ / {
 
        aliases {
                serial2 = &uart2;
+               /* SD Card */
+               mmc0 = &mmc0;
+               /* emmc */
+               mmc1 = &mmc2;
        };
 
        chosen {
```

### TODO : 详细解释为何这样修改就可以固定序号

重新烧录可以看到无论重启多少次，mmc的序号都是固定不变的：

```bash
[    1.244304] Waiting for root device /dev/mmcblk0p2...
[    1.284821] mmc0: host does not support reading read-only switch, assuming write-enable
[    1.294664] mmc0: new high speed SDXC card at address 59b4
[    1.307075] mmcblk0: mmc0:59b4 SD64G 59.4 GiB
[    1.323511]  mmcblk0: p1 p2
```

接下来就是紧急制作一个文件系统：[传送门](https://blog.troy-y.org/2024/09/20/rootfs-%E5%88%B6%E4%BD%9CUbuntu%E6%A0%B9%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/)

制作好根文件系统之后将根文件系统的内容放入rootfs分区，不知道怎么放可以参考这里：[传送门](https://blog.troy-y.org/2024/09/20/sd%E4%B8%8Emmc%E5%88%B7%E5%86%99%E6%8C%87%E5%8D%97/)

重新启动就已经可以进入根文件系统了：

```bash
Ubuntu 22.04 LTS a33-vstar ttyS2

a33-vstar login: root
Password: 
Welcome to Ubuntu 22.04 LTS (GNU/Linux 6.11.0-rc6-00311-g9be470523bd2-dirty armv7l)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Thu Apr  7 19:28:32 UTC 2022 on ttyS2
root@a33-vstar:~# uname
Linux
root@a33-vstar:~# uname -a
Linux a33-vstar 6.11.0-rc6-00311-g9be470523bd2-dirty #6 SMP Fri Sep 20 23:40:53 CST 2024 armv7l armv7l armv7l GNU/Linux
```

