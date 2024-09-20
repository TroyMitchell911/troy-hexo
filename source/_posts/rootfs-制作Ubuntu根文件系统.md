title: '[rootfs]制作Ubuntu根文件系统'
date: '2024-09-20 23:46:54'
updated: '2024-09-20 23:56:53'
tags:
  - ubuntu
categories:
  - kernel
---
**Requirements**

1. An x86_64 machine with Ubuntu or another Linux distribution installed.
2. `debootstrap` tool.
3. Internet connection.
4. Basic knowledge of using the terminal.

**Steps to Create Ubuntu 20.04 Rootfs for ARMhf**

**1. Install Required Tools**

First, ensure that `debootstrap` and `qemu-user-static` are installed. `qemu-user-static` allows you to run ARM binaries on your x86_64 machine.

```
sudo apt update
sudo apt install debootstrap qemu-user-static
```

**2. Create a Directory for the Rootfs**

Create a directory where the root filesystem will be built.

```
mkdir -p ~/ubuntu-armhf-rootfs
```

**3. Run Debootstrap**

Use `debootstrap` to create the root filesystem. Specify the architecture (`armhf`), the Ubuntu release (`focal`), and the target directory.

```
sudo debootstrap --arch=armhf --foreign focal ./ubuntu-armhf-rootfs https://mirrors.bfsu.edu.cn/ubuntu-ports/
```

**4. Copy QEMU Binary**

Copy the `qemu-arm-static` binary into the `usr/bin` directory of the new rootfs to enable emulation.

```
sudo cp /usr/bin/qemu-arm-static ./ubuntu-armhf-rootfs/usr/bin/
```

**5. Chroot into the New Rootfs**

Change root into the new root filesystem to complete the second stage of debootstrap.

```
sudo chroot ./ubuntu-armhf-rootfs
```

**6. Complete Debootstrap Second Stage**

Inside the chroot environment, run the second stage of debootstrap.

```
/debootstrap/debootstrap --second-stage
```

**7. Configure the Rootfs**

Now configure the basic settings of your new root filesystem.

\- **Set the hostname:**

```
echo "ubuntu-armhf" > /etc/hostname
```

\- **Set up the hosts file:**

```
cat <<EOL > /etc/hosts
127.0.0.1   localhost
127.0.1.1   ubuntu-armhf
EOL
```

\- **Create fstab:**

```
cat <<EOL > /etc/fstab
proc            /proc           proc    defaults        0       0
sysfs           /sys            sysfs   defaults        0       0
devpts          /dev/pts        devpts  gid=5,mode=620  0       0
tmpfs           /run            tmpfs   defaults        0       0
tmpfs           /run/lock       tmpfs   nodev,nosuid,noexec 0   0
EOL
```

\- **Set up networking:**

```
cat <<EOL > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
EOL
```

\- **Set the root password:**

```
passwd
```

\- **Create a user:**

```
adduser ubuntu
usermod -aG sudo ubuntu
```

\- **Install essential packages:**

```
apt update
apt install sudo nano ssh net-tools network-manager usbutils
```

**8. Exit the Chroot**

Exit the chroot environment.

```
exit
```

**9. Clean Up**

Remove the `qemu-arm-static` binary from the rootfs.

```
sudo rm ./ubuntu-armhf-rootfs/usr/bin/qemu-arm-static
```