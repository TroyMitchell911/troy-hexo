title: Ubuntu使用命令行修改分辨率
date: '2024-09-12 09:21:23'
updated: '2024-09-12 09:21:25'
tags:
  - ubuntu
---

```bash
$ xrandr
Screen 0: minimum 320 x 200, current 1920 x 1080, maximum 3840 x 2160
HDMI-1 connected primary 1920x1080+0+0 (normal left inverted right x axis y axis) 477mm x 268mm
   1920x1080     60.00*+
   1680x1050     60.00  
   1280x1024     60.00  
   ...
eDP-1 connected (normal left inverted right x axis y axis)
   1366x768      60.00*+
   1280x720      60.00  
   1024x768      60.00  
   ...
$ xrandr --output HDMI-1 --mode 1920x1080
```