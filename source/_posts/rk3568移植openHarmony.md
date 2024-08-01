title: rk3568移植openHarmony
date: '2024-08-01 16:30:28'
updated: '2024-08-01 16:30:30'
tags:
  - kernel
  - openHarmony
categories:
  - kernel
---
## Env
System: Ubuntu 20.04

### Package
安装编译所需要的软件包：

```bash
sudo apt-get update
sudo apt-get install binutils git git-lfs gnupg flex bison gperf build-essential zlib1g-dev \
gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev \
ccache libgl1-mesa-dev libxml2-utils xsltproc unzip m4 bc gnutls-bin python3.8 python3-pip ruby \
openjdk-8-jdk genext2fs libopencv-dev lz4 libssl-dev libncurses5 git-lfs lib32z1-dev zip curl

# 如果libncurses5这个依赖没安装上，执行apt再次安装依赖
sudo apt install libncurses5
```

### Source

如果使用`ssh`方式下载的话需要配置`gitee`的`SSH公钥`。

```bash
repo init -u git@gitee.com:openharmony/manifest.git -b refs/tags/OpenHarmony-v3.1.3-Release --no-repo-verify
repo sync -c
repo forall -c 'git lfs pull'
```

### Toolchains

对于这一步，如果你有开启`proxy`的话，请关闭。


```bash
bash ./build/prebuilts_download.sh
```


## ref
https://doc.embedfire.com/linux/rk356x/OpenHarmony_manual/zh/latest/doc/linux_introduce/ohos-compile.html