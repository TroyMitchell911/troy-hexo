title: VirtualBox错误
tags:
  - linux
---
当我尝试启动虚拟机时遇到了如下错误：

```
Kernel driver not installed (rc=-1908)

The VirtualBox Linux kernel driver is either not loaded or not set up correctly. Please try setting it up again by executing

'/sbin/vboxconfig'

as root.

If your system has EFI Secure Boot enabled you may also need to sign the kernel modules (vboxdrv, vboxnetflt, vboxnetadp, vboxpci) before you can load them. Please see your Linux system's documentation for more information.

where: suplibOsInit what: 3 VERR_VM_DRIVER_NOT_INSTALLED (-1908) - The support driver is not installed. On linux, open returned ENOENT. 
```

尝试执行'/sbin/vboxconfig'：

```bash
❯ sudo /sbin/vboxconfig
vboxdrv.sh: Stopping VirtualBox services.
vboxdrv.sh: Starting VirtualBox services.
vboxdrv.sh: You must sign these kernel modules before using VirtualBox:
  vboxdrv vboxnetflt vboxnetadp
See the documentation for your Linux distribution..
vboxdrv.sh: Building VirtualBox kernel modules.
vboxdrv.sh: failed: Look at /var/log/vbox-setup.log to find out what went wrong.

There were problems setting up VirtualBox.  To re-start the set-up process, run
  /sbin/vboxconfig
as root.  If your system is using EFI Secure Boot you may need to sign the
kernel modules (vboxdrv, vboxnetflt, vboxnetadp, vboxpci) before you can load
them. Please see your Linux system's documentation for more information.
```

查看secure boot是否开启：

```bash
❯ mokutil --sb-state

SecureBoot enabled
```

进入BIOS关闭secure boot后重新执行：

```bash
❯ sudo /sbin/vboxconfig

vboxdrv.sh: Stopping VirtualBox services.
vboxdrv.sh: Starting VirtualBox services.
vboxdrv.sh: Building VirtualBox kernel modules.
vboxdrv.sh: failed: Look at /var/log/vbox-setup.log to find out what went wrong.

There were problems setting up VirtualBox.  To re-start the set-up process, run
  /sbin/vboxconfig
as root.  If your system is using EFI Secure Boot you may need to sign the
kernel modules (vboxdrv, vboxnetflt, vboxnetadp, vboxpci) before you can load
them. Please see your Linux system's documentation for more information.
```

一样的错误，尝试安装编译内核模块所需的软件包。通常需要 dkms 和 kernel-devel后一切正常：

```bash
❯ sudo apt-get install dkms build-essential linux-headers-$(uname -r)
```