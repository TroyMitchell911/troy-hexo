title: Ubuntu22.04多屏闪烁问题
date: '2024-07-24 11:44:16'
updated: '2024-07-24 11:44:19'
tags:
  - linux
  - ubuntu
---
当电脑连接多个显示屏时，只要在副屏打字且光标没有悬浮在主屏幕，主屏幕就会白屏闪烁。

后续发现只有在终端和文件夹出现这个问题，想到安装了`Blur my shell`这个extension，并且指定了这两个app为模糊，在这个extension里面删掉这两个app的配置即可解决。