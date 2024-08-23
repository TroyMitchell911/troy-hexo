title: Rockchip 系列启动流程解读
date: '2024-08-23 12:57:43'
updated: '2024-08-23 12:57:45'
tags:
  - rockchip
categories:
  - kernel
  - rockchip
---

## Soc启动流程

Soc在上电之后，第一个执行的代码是芯片是`BootRom`，通常来说，SoC厂家都会做一个`ROM`在SoC的内部，这个`ROM`很小，里面固化了上电启动的代码（一经固化，永不能改，是芯片做的时候，做进去的）；这部分代码呢，我们管它叫做`BootROM`，也叫作`一级启动程序`。

BootRom需要做的事情：初始化系统，CPU的配置，关闭看门狗，初始化时钟，初始化一些外设（比如 `USB Controller`、`MMC Controller`，`Nand Controller`等）；

`BootROM`的代码除了去初始化硬件环境以外，还需要去外部存储器上面，将接下来可执行的程序读到内存来执行。

但此时`dram`还没有初始化完成，所以并不能直接读取程序到`dram`上执行，这时候芯片内部自带的sram就派上用场了，但`sram`造价高昂，所以通常内存容量较小，只能加载一小段程序到sram运行，这一小段程序只需要负责初始化`dram`，被称作`TPL(Targer Program Loader)`。

`TPL`初始化好`dram`之后将跳会到`BootRom`，`BootRom`再加载一段程序用以将`uboot`和`trust`复制到`dram`并运行，这一小段程序就叫`SPL(Secondary Program Loader)`。

上文提到了一个专业名词叫做`trust`，因为`RK3399`是`ARM64`，所以我们还需要编译·`ATF (ARM Trust Firmware)`，`ATF`主要负责在启动`uboot`之前把CPU从安全的`EL3`切换到`EL2`，然后跳转到`uboot`，并且在内核启动后负责启动其他的CPU。


## Ref

https://www.cnblogs.com/zyly/p/17380243.html
https://www.cnblogs.com/zyly/p/17389525.html#_label0
https://github.com/Caesar-github/docs/tree/master/Common/UBOOT