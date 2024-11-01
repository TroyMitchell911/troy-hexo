title: rk3568刷写openharmony4.1
date: '2024-10-29 16:46:34'
updated: '2024-10-29 16:46:37'
tags:
  - openHarmony
  - rk3568
---
```bash
sudo upgrade_tool di -p parameter.txt
sudo upgrade_tool UL MiniLoaderAll.bin -noreset
sudo upgrade_tool di -u uboot.img && sudo upgrade_tool di -boot_linux boot_linux.img&& sudo upgrade_tool di -system system.img && sudo upgrade_tool di -vendor vendor.img && sudo upgrade_tool di -userdata userdata.img && sudo upgrade_tool di -ramdisk ramdisk.img && sudo upgrade_tool di -resource resource.img && sudo upgrade_tool di -sys-prod sys_prod.img && sudo upgrade_tool di -chip-prod chip_prod.img && sudo upgrade_tool di -eng_system eng_system.img&& sudo upgrade_tool di -updater updater.img && sudo upgrade_tool di -chip_ckm chip_ckm.img
sudo upgrade_tool rd 
```