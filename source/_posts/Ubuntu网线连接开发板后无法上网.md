title: Ubuntu网线连接开发板后无法上网
date: '2024-11-28 21:08:11'
updated: '2024-11-28 21:08:13'
tags:
  - ubuntu
---
```bash
❯ ip route

default via 192.168.8.1 dev enx00e099a751b1 proto static metric 100
default via 192.168.5.1 dev wlp1s0 proto dhcp metric 600
...
```
其中enx00e099a751b1是连接开发板的有线网卡，wlp1s0是无线网卡。
他们两个都走了default的默认路由。但是连接开发板的有线网卡是手动配置的静态IP，肯定不能上网的，如果由enx00e099a751b1去路由流量，肯定就不能上网了。
所以我们要删除这个路由，将enx00e099a751b1只路由192.168.8.x的流量。

```bash
❯ sudo ip route del default via 192.168.8.1 dev enx00e099a751b1
❯ sudo ip route add 192.168.8.0/24 dev enx00e099a751b1
```