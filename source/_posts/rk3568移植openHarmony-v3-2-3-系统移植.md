title: rk3568移植openHarmony v3.2.3---系统移植
date: '2024-08-16 17:32:14'
updated: '2024-08-19 15:34:27'
tags:
  - openHarmony
  - kernel
  - rk3568
  - rockchip
categories:
  - kernel
  - rockchip
---
## 修改build_kernel.sh脚本

- KERNEL_PATCH增加2个，增加了板级patch和jl2101的patch
- 增加CONFIG_PATCH对config文件打补丁
- 增加拷贝设备树到内核临时目录
- 修改了传递给`make-ohos.sh`脚本的参数

```bash
❯ git diff build_kernel.sh
diff --git a/rk3568/kernel/build_kernel.sh b/rk3568/kernel/build_kernel.sh
index 4bd1e65..c205e0b 100755
--- a/rk3568/kernel/build_kernel.sh
+++ b/rk3568/kernel/build_kernel.sh
@@ -23,12 +23,17 @@ export DEVICE_NAME=${7}
 export PRODUCT_COMPANY=${8}
 ENABLE_LTO_O0=${9}
 
+YOUR_BOARD_NAME=your_board_name
+
 KERNEL_SRC_TMP_PATH=${ROOT_DIR}/out/kernel/src_tmp/linux-5.10
 KERNEL_OBJ_TMP_PATH=${ROOT_DIR}/out/kernel/OBJ/linux-5.10
 KERNEL_SOURCE=${ROOT_DIR}/kernel/linux/linux-5.10
 KERNEL_PATCH_PATH=${ROOT_DIR}/kernel/linux/patches/linux-5.10
-KERNEL_PATCH=${ROOT_DIR}/kernel/linux/patches/linux-5.10/rk3568_patch/kernel.patch
+KERNEL_PATCHES="${ROOT_DIR}/kernel/linux/patches/linux-5.10/rk3568_patch/kernel.patch
+		${ROOT_DIR}/kernel/linux/patches/linux-5.10/rk3568_patch/$YOUR_BOARD_NAME.patch
+		${ROOT_DIR}/kernel/linux/patches/linux-5.10/rk3568_patch/add-jl201-drivers.patch"
 KERNEL_CONFIG_FILE=${ROOT_DIR}/kernel/linux/config/linux-5.10/arch/arm64/configs/rk3568_standard_defconfig
+KERNEL_CONFIG_PATCH=${ROOT_DIR}/kernel/linux/patches/linux-5.10/${YOUR_BOARD_NAME}_config.patch
 NEWIP_PATCH_FILE=${ROOT_DIR}/foundation/communication/sfc/newip/apply_newip.sh
 
 rm -rf ${KERNEL_SRC_TMP_PATH}
@@ -46,17 +51,27 @@ cd ${KERNEL_SRC_TMP_PATH}
 bash ${ROOT_DIR}/drivers/hdf_core/adapter/khdf/linux/patch_hdf.sh ${ROOT_DIR} ${KERNEL_SRC_TMP_PATH} ${KERNEL_PATCH_PATH} ${DEVICE_NAME}
 
 #kernel patch
-patch -p1 < ${KERNEL_PATCH}
+for KERNEL_PATCH in $KERNEL_PATCHES
+do
+    patch -p1 < ${KERNEL_PATCH}
+done
 
 #newip
 if [ -f $NEWIP_PATCH_FILE ]; then
 bash $NEWIP_PATCH_FILE ${ROOT_DIR} ${KERNEL_SRC_TMP_PATH} ${DEVICE_NAME} linux-5.10
 fi
 
+#kernel dts
+cp -rf ${3}/kernel/dts/*.dts* ${KERNEL_SRC_TMP_PATH}/arch/arm64/boot/dts/rockchip/
+# jl2101 driver
+cp -rf ${3}/kernel/JL201/jlsemi* ${KERNEL_SRC_TMP_PATH}/drivers/net/phy/
+
 cp -rf ${3}/kernel/logo* ${KERNEL_SRC_TMP_PATH}/
 
 #config
 cp -rf ${KERNEL_CONFIG_FILE} ${KERNEL_SRC_TMP_PATH}/arch/arm64/configs/rockchip_linux_defconfig
+#config patch
+#patch -p1 < ${CONFIG_PATCH}
 
 ramdisk_arg="disable_ramdisk"
 make_ohos_env="GPUDRIVER=mali"
@@ -72,7 +87,7 @@ do
 			;;
 	esac
 done
-eval $make_ohos_env ./make-ohos.sh TB-RK3568X0 $ramdisk_arg ${ENABLE_LTO_O0}
+eval $make_ohos_env ./make-ohos.sh $YOUR_BOARD_NAME $ramdisk_arg ${ENABLE_LTO_O0}
 
 mkdir -p ${2}
```

## 内核补丁准备工作

首先编写`Makefile`实现自动化复制内核，打补丁的操作：

```makefile
# Linux内核目录
KERNEL_SRC_TMP_PATH := <绝对路径>

# OpenHarmony目录相关变量
OHOS_BUILD_HOME := <绝对路径>
DEVICE_NAME := rk3568
KERNEL_VERSION := linux-5.10
KERNEL_SRC_PATH := $(OHOS_BUILD_HOME)/kernel/linux/${KERNEL_VERSION}
DEVICE_PATCH_DIR := $(OHOS_BUILD_HOME)/kernel/linux/patches/${KERNEL_VERSION}/$(DEVICE_NAME)_patch
DEVICE_PATCH_FILE := $(DEVICE_PATCH_DIR)/kernel.patch
HDF_PATCH_FILE := $(DEVICE_PATCH_DIR)/hdf.patch
KERNEL_PATCH_PATH := $(OHOS_BUILD_HOME)/kernel/linux/patches/${KERNEL_VERSION}

.PHONY: all clean copy_kernel copy_device
copy_kernel:
	rm -rf $(KERNEL_SRC_TMP_PATH);mkdir -p $(KERNEL_SRC_TMP_PATH);cp -arfL $(KERNEL_SRC_PATH)/* $(KERNEL_SRC_TMP_PATH)
    cp $(OHOS_BUILD_HOME)/kernel/linux/config/${KERNEL_VERSION}/arch/arm64/configs/rk3568_standard_defconfig $(KERNEL_SRC_TMP_PATH)/arch/arm64/configs/rockchip_linux_defconfig

put_hdf_patch:
	cd $(KERNEL_SRC_TMP_PATH) && patch -p1 < ${HDF_PATCH_FILE}
ln_hdf_repos:
	cd $(KERNEL_SRC_TMP_PATH) && ln -sf ${OHOS_BUILD_HOME}/drivers/hdf_core/adapter/khdf/linux    drivers/hdf/khdf
	cd $(KERNEL_SRC_TMP_PATH) && ln -sf ${OHOS_BUILD_HOME}/drivers/hdf_core/framework             drivers/hdf/framework
	cd $(KERNEL_SRC_TMP_PATH) && ln -sf ${OHOS_BUILD_HOME}/drivers/hdf_core/framework/include     include/hdf
copy_external_compents:
	cd $(KERNEL_SRC_TMP_PATH) && cp -arfL ${OHOS_BUILD_HOME}/third_party/bounds_checking_function  ./
	cd $(KERNEL_SRC_TMP_PATH) && mkdir -p drivers/hdf/
	cd $(KERNEL_SRC_TMP_PATH) && cp -arfL ${OHOS_BUILD_HOME}/device/soc/hisilicon/common/platform/wifi         drivers/hdf/
	cd $(KERNEL_SRC_TMP_PATH) && cp -arfL ${OHOS_BUILD_HOME}/third_party/FreeBSD/sys/dev/evdev     drivers/hdf/
	    
patch_hdf_all:
	$(OHOS_BUILD_HOME)/drivers/hdf_core/adapter/khdf/linux/patch_hdf.sh $(OHOS_BUILD_HOME) $(KERNEL_SRC_TMP_PATH) $(KERNEL_PATCH_PATH) $(DEVICE_NAME)
	#make put_hdf_patch
	#make ln_hdf_repos
	#make copy_external_compentsls

patch_kernel:
	cd $(KERNEL_SRC_TMP_PATH) && patch -p1 < $(DEVICE_PATCH_FILE)
copy_config:
	cp -rf $(KERNEL_CONFIG_PATH)/. $(KERNEL_SRC_TMP_PATH)/

all:
	make clean
	make copy_kernel
	make patch_hdf_all
	make patch_kernel

clean:
	rm -rf $(KERNEL_SRC_TMP_PATH)

```

依次执行以下命令：

```bash
make copy_kernel
make patch_kernel
find ./* -name "*.orig" | xargs rm -rf
make patch_hdf

创建设备树目录并将已经适配好`Linux`系统的设备树文件拷贝进来(这一步是为了`build_kernel.sh`将此文件夹中的设备树拷贝到内核临时目录的设备树文件夹中)：

```bash
mkdir -p device/board/hihope/rk3568/kernel/dts
cp rk3568-xxx.dts device/board/hihope/rk3568/kernel/dts
```

创建git仓库并且提交：

```bash
git init && git add -A && git commit -m "First"
```

## 增加内核补丁

进入内核临时目录后修改`make-ohos.h`：
    - 修改了DTB变量
    - 修改了model_list

```bash
diff --git a/make-ohos.sh b/make-ohos.sh
index 4dc437165..704e8c78f 100755
--- a/make-ohos.sh
+++ b/make-ohos.sh
@@ -14,7 +14,7 @@ MAKE="make LLVM=1 LLVM_IAS=1 CROSS_COMPILE=../../../../prebuilts/gcc/linux-x86/a
 BUILD_PATH=boot_linux
 EXTLINUX_PATH=${BUILD_PATH}/extlinux
 EXTLINUX_CONF=${EXTLINUX_PATH}/extlinux.conf
-TOYBRICK_DTB=toybrick.dtb
+YOUR_BOARD_NAME_DTB=rk3568-your_board_name.dtb
 if [ ${KBUILD_OUTPUT} ]; then
 	OBJ_PATH=${KBUILD_OUTPUT}/
 fi
@@ -26,8 +26,7 @@ ID_DTB=4
 ID_IMAGE=5
 ID_CONF=6
 model_list=(
-	"TB-RK3568X0   arm64 0xfe660000 rk3568-toybrick-x0-linux  Image rockchip_linux_defconfig"
-	"TB-RK3568X10  arm64 0xfe660000 rk3568-toybrick-x10-linux Image rockchip_linux_defconfig"
+	"your_board_name   arm64 0xfe660000 rk3568-your_board_name  Image rockchip_linux_defconfig"
 )
 
 
@@ -49,7 +48,7 @@ function make_extlinux_conf()
 	
 	echo "label rockchip-kernel-5.10" > ${EXTLINUX_CONF}
 	echo "	kernel /extlinux/${image}" >> ${EXTLINUX_CONF}
-	echo "	fdt /extlinux/${TOYBRICK_DTB}" >> ${EXTLINUX_CONF}
+	echo "	fdt /extlinux/${YOUR_BOARD_NAME_DTB}" >> ${EXTLINUX_CONF}
 	cmdline="append earlycon=uart8250,mmio32,${uart} root=PARTUUID=614e0000-0000-4b53-8000-1d28000054a9 rw rootwait rootfstype=ext4"
 	echo "  ${cmdline}" >> ${EXTLINUX_CONF}
 }
@@ -118,7 +117,7 @@ function make_boot_linux()
 	fi
 	make_extlinux_conf ${dtb_path} ${uart} ${image}
 	cp -f ${OBJ_PATH}arch/${arch}/boot/${image} ${EXTLINUX_PATH}/
-	cp -f ${OBJ_PATH}${dtb_path}/${dtb}.dtb ${EXTLINUX_PATH}/${TOYBRICK_DTB}
+	cp -f ${OBJ_PATH}${dtb_path}/${dtb}.dtb ${EXTLINUX_PATH}/${YOUR_BOARD_NAME_DTB}
 	cp -f logo*.bmp ${BUILD_PATH}/
 	if [ "enable_ramdisk" != "${ramdisk_flag}" ]; then
 		make_ext2_image
```

修改`arch/arm64/boot/dts/rockchip/Makefile`加入你的板子。

执行以下命令提交：

```bash
git add -A && git commit -m "something you like"
```

执行以下命令生成patch:

```bash
git format-patch --subject-prefix='PATCH' -n -s -1
```

将这个`patch`移动到`kernel/linux/patches/linux-5.10/rk3568_patch/$YOUR_BOARD_NAME.patch`下，`YOUR_BOARD_NAME`对应你`build_kernel.sh`中定义的。

根据自己的需要修改`arch/arm64/configs/rockchip_linux_defconfig`后提交，随后生成补丁，移动到`build_kernel.sh`中设置的`CONFIG_PATCH_FILE`。

回到主目录，删除`out/kernel`文件夹后重新编译，即可在`out/rk3568/packages/phone/images`目录下看到输出文件。

```bash
rm -rf out/kernel
./build.sh --product-name rk3568 --ccache

❯ cd out/rk3568/packages/phone/images
❯ ls
boot_linux.img  config.cfg         parameter.txt  resource.img  system.img  updater.img   vendor.img
chip_prod.img   MiniLoaderAll.bin  ramdisk.img    sys_prod.img  uboot.img   userdata.img
```