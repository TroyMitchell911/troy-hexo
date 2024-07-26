title: rk3568刷写openharmony3.2
date: '2024-07-25 20:15:03'
updated: '2024-07-25 20:15:05'
tags:
  - linux
  - kernel
categories:
  - kernel
---
```bash
sudo upgrade_tool UL MiniLoaderAll.bin -noreset
sudo upgrade_tool di -u uboot.img && sudo upgrade_tool di -boot_linux boot_linux.img&& sudo upgrade_tool di -system system.img && sudo upgrade_tool di -vendor vendor.img && sudo upgrade_tool di -userdata userdata.img && sudo upgrade_tool di -ramdisk ramdisk.img && sudo upgrade_tool di -resource resource.img && sudo upgrade_tool di -sys-prod sys_prod.img && sudo upgrade_tool di -chip-prod chip_prod.img
sudo upgrade_tool rd
```