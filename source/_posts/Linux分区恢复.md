title: Linux分区恢复
date: '2024-09-13 14:43:18'
updated: '2024-09-13 14:43:20'
tags:
  - ubuntu
categories:
  - kernel
---
在删除U盘分区的时候，忘了插U盘，直接把Ubuntu的efi分区删掉了。

直接原地红温...

不过还好删除的是分区不是数据，还有的救...

**注意这时候千万不要重启系统！！！**

查看分区挂载：

```bash
❯ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
...
nvme0n1     259:0    0 953.9G  0 disk 
└─nvme0n1p2 259:2    0 953.4G  0 part /
```

已经看不见p1了，但是！

执行如下命令安装众神之父`testdisk`:

```bash
❯ sudo apt install testdisk 
```

这个工具非常好用，可以自定义检测分区表，并且检测丢失的分区：

```bash
❯ sudo testdisk /dev/nvme0n1
```

进入如下界面, 这里是选择磁盘，我只有一个盘，所以就直接按enter了：

[![2024-09-13-14-24-41.png](https://i.postimg.cc/26f8LZRh/2024-09-13-14-24-41.png)](https://postimg.cc/DWBKtmzw)

选择磁盘后需要选择你的磁盘分区表类型，Intel是自动选择，按道理应该是GPT分区，但是不太自信，所以选择了Intel进入：

[![2024-09-13-14-24-47.png](https://i.postimg.cc/nV49cbkR/2024-09-13-14-24-47.png)](https://postimg.cc/qg7vjFWy)

这里显示检测到了GPT分区，直接按Enter进入：

[![2024-09-13-14-25-08.png](https://i.postimg.cc/qBxgHcJW/2024-09-13-14-25-08.png)](https://postimg.cc/jCCRz7Jh)

在如下界面中直接选中Analyse回车：

[![2024-09-13-14-48-06.png](https://i.postimg.cc/J0z8LkKW/2024-09-13-14-48-06.png)](https://postimg.cc/Kkww74d9)

随后检测到分区结构如下，直接选择Quick Search回车：

[![2024-09-13-14-51-29.png](https://i.postimg.cc/qMgrD2zs/2024-09-13-14-51-29.png)](https://postimg.cc/bsX4sD0r)

接下来检测到一个可恢复分区FAT，回车进入后选择Write，选择Write只会重写你的分区信息，而不会删除你的分区数据，所以大可放心。

选择Write之后会询问你是否确认，输入Y即可。

[![2024-09-13-14-25-12.png](https://i.postimg.cc/3wjQ0BcT/2024-09-13-14-25-12.png)](https://postimg.cc/VSNV3jwG)

写入之后重启系统，就可以看到心心念念的efi了。

```bash
❯ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
...
nvme0n1     259:0    0 953.9G  0 disk 
├─nvme0n1p1 259:1    0   512M  0 part /boot/efi
└─nvme0n1p2 259:2    0 953.4G  0 part /
```
