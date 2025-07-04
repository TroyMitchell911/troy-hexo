title: 使用pxelinux启动内核
date: '2024-09-06 10:33:32'
updated: '2024-09-06 10:33:34'
tags:
  - linux
categories:
  - uboot
---
## Env

Board: BPI-F3 based on k1 of SpaceMit

**Note: 本文默认已经在主机待建成功tftp服务**

## Content

### 设置IP

在主机上查询`ip`:

```bash
❯ ifconfig
enx00e099a751b1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.230.28  netmask 255.255.255.0  broadcast 192.168.230.255
        inet6 fe80::7d95:76be:c484:17b6  prefixlen 64  scopeid 0x20<link>
        ether 00:e0:99:a7:51:b1  txqueuelen 1000  (以太网)
        RX packets 223389  bytes 75866217 (75.8 MB)
        RX errors 0  dropped 94  overruns 0  frame 0
        TX packets 196725  bytes 223909969 (223.9 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

得到主机`ip`是`192.168.230.28`，板子`ip`确保在一个局域网内就好。

接下来进入`uboot`设置`ip`：

```bash
=> setenv ipaddr 192.168.230.4
=> setenv serverip 192.168.230.28
=> [ 723.486] printenv ipaddr 
ipaddr=192.168.230.4
=> [ 727.153] printenv serverip
serverip=192.168.230.28
=> saveenv
```

### 配置pxe文件

在主机上进入`tftp`文件夹，目录结构如下：

```bash
❯ tree
.
├── Image
├── k1-x_deb1.dtb
└── pxelinux.cfg
    └── 01-fe-fe-fe-81-b4-a8

1 directory, 3 files
```


其中`pxelinux.cfg`是存放`pxe配置文件`的文件夹，由于`TFTP`服务可能会被多个开发板使用，因此`PXE配置文件`的名称取决于`U-boot`参数(板的硬件地址/ IP地址)。

优先级最高的是`mac`地址，在`uboot`中查看`mac`地址：

```bash
=> [ 898.685] printenv ethaddr 
ethaddr=FE:FE:FE:81:B4:A8
```

所以`cfg`的名字就应该是:  `01-fe-fe-fe-81-b4-a8`

这个`01`我也不知道什么意思，反正必须加上。

接下来就是修改 `01-fe-fe-fe-81-b4-a8`这个文件了，文件内容格式跟[extlinux](https://blog.troy-y.org/2024/08/26/%E9%87%8E%E7%81%ABuboot%E4%BD%BF%E7%94%A8extboot%E5%90%AF%E5%8A%A8%E5%86%85%E6%A0%B8%E6%B5%81%E7%A8%8B/#cfg%E6%96%87%E4%BB%B6)是一模一样的，这里不再赘述。

文件内容如下：

```bash
❯ cat pxelinux.cfg/01-fe-fe-fe-81-b4-a8
default linux

label linux
	kernel Image
	fdt k1-x_deb1.dtb
	append earlycon=sbi earlyprintk quiet splash plymouth.ignore-serial-consoles plymouth.prefer-fbcon console=ttyS0,115200 loglevel=8 clk_ignore_unused swiotlb=65536 rdinit=/init workqueue.default_affinity_scope=system root=/dev/mmcblk2p6 rootwait rootfstype=ext4
```

## pxe启动

在`uboot`中执行如下命令：

```bash
pxe get
pxe boot
```

经过以上命令按道理来说就可以启动内核了，但是发现启动到一半卡死了，看`log`发现貌似是设备树没有加载，他使用了一个地址`0x7deb2e10`的设备树：

```
=> [1976.087] pxe get
missing environment variable: pxeuuid
[1978.023] Retrieving file: pxelinux.cfg/01-fe-fe-fe-81-b4-a8
[1978.171] ethernet@cac80000 Waiting for PHY auto negotiation to complete...... done
[1981.637] emac_adjust_link link:1 speed:1000 duplex:full
[1981.659] Using ethernet@cac80000 device
[1981.660] TFTP from server 192.168.230.28; our IP address is 192.168.230.4
Filename 'pxelinux.cfg/01-fe-fe-fe-81-b4-a8'.
Load address: 0xc200000
Loading: #
         [1981.682] 39.1 KiB/s
done
Bytes transferred = 322 (142 hex)
[1981.685] Config file '<NULL>' found
=> [1982.858] pxe boot
1:      linux
[1984.100] Retrieving file: Image
[1984.244] ethernet@cac80000 Waiting for PHY auto negotiation to complete...... done
[1987.710] emac_adjust_link link:1 speed:1000 duplex:full
[1987.732] Using ethernet@cac80000 device
[1987.733] TFTP from server 192.168.230.28; our IP address is 192.168.230.4
Filename 'Image'.
Load address: 0x11000000
Loading: #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #####
         [2011.448] 1.4 MiB/s
done
Bytes transferred = 34423808 (20d4400 hex)
[2011.451] append: earlycon=sbi earlyprintk quiet splash plymouth.ignore-serial-consoles plymouth.prefer-fbcon console=ttyS0,115200 loglevel=8 clk_igno
re_unused swiotlb=65536 rdinit=/init workqueue.default_affinity_scope=system root=/dev/mmcblk2p6 rootwait rootfstype=ext4
[2011.475] Moving Image from 0x11000000 to 0x200000, end=2358000
[2011.493] ## Flattened Device Tree blob at 7deb2e10
[2011.495]    Booting using the fdt blob at 0x7deb2e10
[2011.500]    Loading Device Tree to 000000007dd8f000, end 000000007dd9cf27 ... OK
```
这就很奇怪了，但运气使然，在`uboot`中发现了这么一个变量：

```bash
=>  printenv fdtcontroladdr 
fdtcontroladdr=7deb2e10
```

这和刚才在内存中加载的设备树的地址一模一样，但尝试将他清空再执行`pxe`，same thing，看来有点运气，但不多。

经过查询资料发现，`pxe`会将`conf`指定的`Image`加载到`kernel_addr_r`处，将`dtb`加载到`fdt_addr_r`处，那么是否我们的`uboot`默认只设置了`kernel`的地址而没有`dtb`的地址，在`uboot`中输入以下指令：

```bash
=> [  58.262] printenv kernel_addr_r 
kernel_addr_r=0x11000000
=> [  59.428] printenv dtb_addr_r    
## Error: "dtb_addr_r" not defined
```

果然..，增加`fdt_addr_r`变量：

```bash
=> printenv dtb_addr 
dtb_addr=0x31000000
=> setenv fdt_addr_r 0x31000000
=> saveenv
Saving Environment to MMC... Writing to MMC(2)... OK
```
此时再通过`pxe`启动就没有任何问题了。

## Ref

https://blog.csdn.net/weixin_35808698/article/details/117274748
