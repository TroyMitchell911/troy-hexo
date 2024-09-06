title: 使用pxelinux启动内核
---
## Env

Board: BPI-F3 based on k1 of SpaceMit

** Note: 本文默认已经在主机待建成功tftp服务 **

## Content

### 设置IP

在主机上查询ip:

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

得到主机ip是192.168.230.28，板子ip确保在一个局域网内就好。

接下来进入uboot设置ip：

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

在主机上进入tftp文件夹，目录结构如下：

```bash
❯ tree
.
├── Image
├── k1-x_deb1.dtb
└── pxelinux.cfg
    └── 01-fe-fe-fe-81-b4-a8

1 directory, 3 files
```


其中pxelinux.cfg是存放pxe配置文件的文件夹，由于TFTP服务可能会被多个开发板使用，因此“ PXE配置文件”的名称取决于U-boot参数(板的硬件地址/ IP地址)。

优先级最高的是mac地址，在uboot中查看mac地址：

```bash
=> [ 898.685] printenv ethaddr 
ethaddr=FE:FE:FE:81:B4:A8
```

所以cfg的名字就应该是:  01-fe-fe-fe-81-b4-a8

这个01我也不知道什么意思，反正必须加上。

接下来就是修改 01-fe-fe-fe-81-b4-a8这个文件了，文件内容格式跟[extlinux](https://blog.505218.xyz/2024/08/26/%E9%87%8E%E7%81%ABuboot%E4%BD%BF%E7%94%A8extboot%E5%90%AF%E5%8A%A8%E5%86%85%E6%A0%B8%E6%B5%81%E7%A8%8B/#cfg%E6%96%87%E4%BB%B6)是一模一样的，这里不再赘述。