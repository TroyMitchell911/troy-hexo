title: rk3568移植openHarmony v3.2.3---编译流程分析
date: '2024-08-16 14:39:10'
updated: '2024-08-16 14:39:11'
tags:
  - openHarmony
  - rk3568
  - rockchip
  - kernel
categories:
  - kernel
  - rockchip
---
## 目录分析

在`OpenHarmony`编译过程中，内核源码是不变的，内核源码保存在`kernel/linux/linux-5.10/`下，并且永远不被修改。

在编译内核前，会将内核源码复制到内核临时源码目录`out/kernel/src_tmp/linux-5.10/`下，然后再以打补丁的方式进行修改。

在编译命令`./build.sh --product-name rk3568 --ccache`中，用`--product`命令指定了`rk3568`。

通过查阅文档得知，这个`product`参数是在`vendor`文件夹中所指定的。

```bash

❯ vim vendor/hihope/rk3568/config.json

{
  "product_name": "rk3568",
  "device_company": "rockchip",
  "device_build_path": "device/board/hihope/rk3568",
  "target_cpu": "arm",
  "type": "standard",
  "version": "3.0",
  "board": "rk3568",
    ...
}
```

在该配置文件中指定了`product_name`为`rk3568`的目标的编译路径是`device/board/hihope/rk3568`。

```bash
❯ cd device/board/hihope/rk3568
❯ ls
audio_drivers  bootanimation  BUILD.gn  camera  cfg  config.gni  device.gni  distributedhardware  kernel  loader  ohos.build  startup  updater  wifi
```

目标的编译路径下面有很多文件，我们关心的是内核移植，所以只需要看`kernel`文件夹。

```bash
❯ cd kernel && ls
BUILD.gn  build_kernel.sh  dts  JL201  logo.bmp  logo_kernel.bmp  ov8858.c
```

先从`BUILD.gn`文件开始看：

```bash
❯ cat BUILD.gn
···
import("//build/config/clang/clang.gni")
import("//build/ohos.gni")
kernel_build_script_dir = "//kernel/linux/linux-5.10"
kernel_source_dir = "//kernel/linux/linux-5.10"

action("kernel") {
  script = "build_kernel.sh"
  sources = [ kernel_source_dir ]

  product_path = "vendor/$product_company/$product_name"
  outputs = [ "$root_build_dir/../kernel/src_tmp/linux-5.10/boot_linux" ]
  args = [
    rebase_path(kernel_build_script_dir, root_build_dir),
    rebase_path("$root_build_dir/packages/phone/images"),
    rebase_path("//device/board/hihope/$device_name"),
    product_path,
    rebase_path("$root_build_dir/../.."),
    device_company,
    device_name,
    product_company,
  ]
···
}
```

可以看到当构建流执行到这里的时候会直接执行`build_kernel.sh`这个脚本，并且传入一些参数。

## 编译流程

查看`build_kernel.sh`这个脚本：

```bash
set -e

pushd ${1}
ROOT_DIR=${5}
export PRODUCT_PATH=${4}
export DEVICE_COMPANY=${6}
export DEVICE_NAME=${7}
export PRODUCT_COMPANY=${8}
ENABLE_LTO_O0=${9}

KERNEL_SRC_TMP_PATH=${ROOT_DIR}/out/kernel/src_tmp/linux-5.10
KERNEL_OBJ_TMP_PATH=${ROOT_DIR}/out/kernel/OBJ/linux-5.10
KERNEL_SOURCE=${ROOT_DIR}/kernel/linux/linux-5.10
KERNEL_PATCH_PATH=${ROOT_DIR}/kernel/linux/patches/linux-5.10
KERNEL_PATCH=${ROOT_DIR}/kernel/linux/patches/linux-5.10/rk3568_patch/kernel.patch
KERNEL_CONFIG_FILE=${ROOT_DIR}/kernel/linux/config/linux-5.10/arch/arm64/configs/rk3568_standard_defconfig
NEWIP_PATCH_FILE=${ROOT_DIR}/foundation/communication/sfc/newip/apply_newip.sh

rm -rf ${KERNEL_SRC_TMP_PATH}
mkdir -p ${KERNEL_SRC_TMP_PATH}

rm -rf ${KERNEL_OBJ_TMP_PATH}
mkdir -p ${KERNEL_OBJ_TMP_PATH}
export KBUILD_OUTPUT=${KERNEL_OBJ_TMP_PATH}

cp -arf ${KERNEL_SOURCE}/* ${KERNEL_SRC_TMP_PATH}/

cd ${KERNEL_SRC_TMP_PATH}

#HDF patch
bash ${ROOT_DIR}/drivers/hdf_core/adapter/khdf/linux/patch_hdf.sh ${ROOT_DIR} ${KERNEL_SRC_TMP_PATH} ${KERNEL_PATCH_PATH} ${DEVICE_NAME}

#kernel patch
patch -p1 < ${KERNEL_PATCH}

#newip
if [ -f $NEWIP_PATCH_FILE ]; then
bash $NEWIP_PATCH_FILE ${ROOT_DIR} ${KERNEL_SRC_TMP_PATH} ${DEVICE_NAME} linux-5.10
fi

cp -rf ${3}/kernel/logo* ${KERNEL_SRC_TMP_PATH}/

#config
cp -rf ${KERNEL_CONFIG_FILE} ${KERNEL_SRC_TMP_PATH}/arch/arm64/configs/rockchip_linux_defconfig

ramdisk_arg="disable_ramdisk"
make_ohos_env="GPUDRIVER=mali"
for i in "$@" 
do
	case $i in
		enable_ramdisk)
			ramdisk_arg=enable_ramdisk
			;;
		enable_mesa3d)
			make_ohos_env="GPUDRIVER=mesa3d"
			python ${ROOT_DIR}/third_party/mesa3d/ohos/modifyDtsi.py ${KERNEL_SRC_TMP_PATH}/arch/arm64/boot/dts/rockchip/rk3568.dtsi
			;;
	esac
done
eval $make_ohos_env ./make-ohos.sh TB-RK3568X0 $ramdisk_arg ${ENABLE_LTO_O0}

mkdir -p ${2}

if [ "enable_ramdisk" != "${10}" ]; then
	cp ${KERNEL_OBJ_TMP_PATH}/boot_linux.img ${2}/boot_linux.img
fi
cp ${KERNEL_OBJ_TMP_PATH}/resource.img ${2}/resource.img
cp ${3}/loader/parameter.txt ${2}/parameter.txt
cp ${3}/loader/MiniLoaderAll.bin ${2}/MiniLoaderAll.bin
cp ${3}/loader/uboot.img ${2}/uboot.img
cp ${3}/loader/config.cfg ${2}/config.cfg
popd

../kernel/src_tmp/linux-5.10/make-boot.sh ..
```

### 变量解释

    - KERNEL_SRC_TMP_PATH: 源码临时路径，在编译流程中，内核的原源码是不变的，需要将内核源码复制到这个路径下，然后对这个路径下的内核源码进行打补丁。
    - KERNEL_OBJ_TMP_PATH: 目标临时路径。
    - KERNEL_SOURCE: 内核原源码路径
    - KERNEL_PATCH_PATH: 内核补丁路径
    - KERNEL_PATCH: 内核补丁文件
    - KERNEL_CONFIG_FILE: 内核配置文件
    - NEWIP_PATCH_FILE: newip补丁文件

### 脚本解释

将内核源码复制到内核临时源码目录并且进入：

```bash
rm -rf ${KERNEL_SRC_TMP_PATH}
mkdir -p ${KERNEL_SRC_TMP_PATH}

rm -rf ${KERNEL_OBJ_TMP_PATH}
mkdir -p ${KERNEL_OBJ_TMP_PATH}
export KBUILD_OUTPUT=${KERNEL_OBJ_TMP_PATH}

cp -arf ${KERNEL_SOURCE}/* ${KERNEL_SRC_TMP_PATH}/

cd ${KERNEL_SRC_TMP_PATH}
```

对内核源码进行打补丁：

```bash
#HDF patch
bash ${ROOT_DIR}/drivers/hdf_core/adapter/khdf/linux/patch_hdf.sh ${ROOT_DIR} ${KERNEL_SRC_TMP_PATH} ${KERNEL_PATCH_PATH} ${DEVICE_NAME}

#kernel patch
patch -p1 < ${KERNEL_PATCH}
```

其中HDF的补丁调用到了`drivers/hdf_core/adapter/khdf/linux/patch_hdf.sh`这个脚本，脚本参数如下：
    - OH工程根目录，绝对路径
    - 内核临时目录，绝对路径
    - 内核补丁路径，绝对路径
    - 设备名称

`patch_hdf.sh`脚本对于补丁的搜索：

```bash
function put_hdf_patch()
{
    HDF_PATCH_FILE=${KERNEL_PATCH_PATH}/${DEVICE_NAME}_patch/hdf.patch
    if [ ! -e "${HDF_PATCH_FILE}" ]
    then
	    HDF_PATCH_FILE=${KERNEL_PATCH_PATH}/${HDF_COMMON_PATCH}_patch/hdf.patch
    fi
    patch -p1 < $HDF_PATCH_FILE
}
```

可以看到此脚本是根据补丁路径+设备名称找到`hdf.patch`这个补丁。

拷贝开机logo和配置文件：

```bash
cp -rf ${3}/kernel/logo* ${KERNEL_SRC_TMP_PATH}/

#config
cp -rf ${KERNEL_CONFIG_FILE} ${KERNEL_SRC_TMP_PATH}/arch/arm64/configs/rockchip_linux_defconfig
```

创建env和执行编译脚本：

```bash
ramdisk_arg="disable_ramdisk"
make_ohos_env="GPUDRIVER=mali"
for i in "$@" 
do
	case $i in
		enable_ramdisk)
			ramdisk_arg=enable_ramdisk
			;;
		enable_mesa3d)
			make_ohos_env="GPUDRIVER=mesa3d"
			python ${ROOT_DIR}/third_party/mesa3d/ohos/modifyDtsi.py ${KERNEL_SRC_TMP_PATH}/arch/arm64/boot/dts/rockchip/rk3568.dtsi
			;;
	esac
done
eval $make_ohos_env ./make-ohos.sh TB-RK3568X0 $ramdisk_arg ${ENABLE_LTO_O0}
```

`make-ohos.sh`脚本在内核临时源码目录，`TB-RK3568X0`是选择设备树的。

`make-ohos.sh`脚本中关于设备树如下：

```bash
···
model_list=(
	"TB-RK3568X0   arm64 0xfe660000 rk3568-xxx  Image rockchip_linux_defconfig"
)

···
```

其中`TB-RK3568X0 `是设备树目标名字， `rk3568-xxx`是具体的设备树名字，注意，没有后缀。

回到`build_kernel.sh`，创建编译文件目录并且拷贝过去：

```bash
# $2 = rebase_path("$root_build_dir/packages/phone/images"),
# $3 = rebase_path("//device/board/hihope/$device_name"),


mkdir -p ${2}

if [ "enable_ramdisk" != "${10}" ]; then
	cp ${KERNEL_OBJ_TMP_PATH}/boot_linux.img ${2}/boot_linux.img
fi
cp ${KERNEL_OBJ_TMP_PATH}/resource.img ${2}/resource.img
cp ${3}/loader/parameter.txt ${2}/parameter.txt
cp ${3}/loader/MiniLoaderAll.bin ${2}/MiniLoaderAll.bin
cp ${3}/loader/uboot.img ${2}/uboot.img
cp ${3}/loader/config.cfg ${2}/config.cfg
popd
```

执行生成`boot_linux.img`脚本：

```bash
../kernel/src_tmp/linux-5.10/make-boot.sh ..
```