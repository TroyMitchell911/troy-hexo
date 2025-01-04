title: linux挂载nfs根文件系统失败
date: '2025-01-04 21:54:56'
updated: '2025-01-04 21:54:58'
tags:
  - kernel
  - linux
categories:
  - kernel
  - network
---
挂在nfs根文件系统时出现如下报错：

```bash
[    1.939157] VFS: Cannot open root device "nfs" or unknown-block(0,255): error -6
[    1.946989] Please append a correct "root=" boot option; here are the available partitions:
[    1.955765] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,255)
[    1.964493] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,255) ]---
```

根据内核日志来看`bootargs`是设置正确的，经过多方排查是以下配置没开启：

```bash
CONFIG_ROOT_NFS
```