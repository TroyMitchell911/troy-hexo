title: ubuntu安装hdc工具
date: '2024-08-19 11:14:17'
updated: '2024-08-19 11:14:19'
tags:
  - openHarmony
  - ubuntu
  - linux
---
首先打开链接：https://ci.openharmony.cn/workbench/cicd/dailybuild/dailylist

选择你的openharmony版本，找到对应的系统， 是standard还是Small根据需要选择。

之后选择下载全量包，打开全量包压缩包后可在toolchains中看到hdc，需要注意的是，toolchains整个文件夹都需要解压出来，因为so文件的原因，不能单独执行hdc。

解压完成后添加到~/.bashrc中的PATH变量即可。

**Ref：**

https://docs.openharmony.cn/pages/v4.1/zh-cn/device-dev/subsystems/subsys-toolchain-hdc-guide.md
https://docs.openharmony.cn/pages/v4.1/zh-cn/application-dev/dfx/hdc.md
https://developer.huawei.com/consumer/cn/blog/topic/03137966529669104