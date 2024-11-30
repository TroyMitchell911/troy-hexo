title: Linux命令行运行clash-verge
date: '2024-11-29 21:04:49'
updated: '2024-11-29 21:04:51'
tags:
  - ubuntu
---
由于太懒，不想配置clash core或者mihomo，就下了个clash verge，配置好之后断掉显示器启动电脑却发现clash-verge没有运行。

在Linux命令行中（tty, 没有display）是无法运行clash-verge的，如果直接运行，会出现：

```bash
❯ clash-verge

(clash-verge:4357): Gtk-WARNING **: 20:22:40.304: cannot open display:
```

使用虚拟显示器运行：

```bash
❯ sudo apt install xvfb
❯ Xvfb :1 -screen 0 1024x768x16 &
❯ export DISPLAY=:1 
❯ clash-verge &
```

此时还不能科学，需要等待1到3秒左右，等待完成后查询端口是否监听成功（verge监听7897）：

```bash
❯ netstat -tuln | grep 7897

tcp        0      0 127.0.0.1:7897          0.0.0.0:*               LISTEN
udp        0      0 127.0.0.1:7897          0.0.0.0:*
```
