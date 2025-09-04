---
title: openmpi[1] 什么是openmpi
date: 2025-09-04 09:57:36
tags:
  - openmpi
categories:
  - openmpi
---

## 什么是openmpi?

MPI是一套通信标准，由MPI Forum创建并维护（MPI Forum是一个开放组织，由工业、学术界的高性能计算领域的专家组成）.
MPI是这样一种API：
- 可移植
- 高性能的IPC通信

MPI一般作为一个消息传递的中间件，上层应用程序可以通过调用MPI接口来执行消息传递。
MPI定义了一系列与平台无关的、高度抽象的API接口，用于进程之间的消息传递(这两个进程之间可以在不同主机，但主机之间需要有一个通信的通道, 比如eth)举一个最简单的例子，进程X是发送进程，只需提供消息内容（例如一个双精度数组）以及另一个接收进程的标识（例如进程Y），同时接收进程Y只需提供发送进程的标识（例如进程X），消息就可以从X传递给Y。
注意这个例子中，没有建立连接、没有字节流的转换、没有网络地址的交换，MPI将这些细节都抽象封装了起来，不仅仅是隐藏了复杂性，而且使应用程序能够兼容不同的平台、硬件以及网络类型。
MPI提供的通信模式：

- point to pint
- collective
- broadcast

## openmpi的功能和特性

在这里我并不想讨论MPI的标准一致性，所以让我们跳过。

### 模块化组件架构(MCA)

- 项目分层：OPAL（可移植性层）、OMPI（MPI API）、OSHMEM（OpenSHMEM API）
- 框架系统：BTL（字节传输层）、PML（点对点消息层）、Coll（集合操作）等
- 组件插件：支持运行时和编译时的组件选择

关于更详细的框架系统我会在后面的章节介绍。

### 多种传输协议的支持

得益于框架系统的分层架构，openmpi通过btl或coll层支持了多重传输协议，包括：

- self: 用于同一节点同一进程内的通信
- 共享内存：用于同一节点内进程间高效通信
- TCP/IP：通用网络传输
- 高性能网络：InfiniBand、Omni-Path、以太网等

### 加速器框架

新的加速器框架支持 CUDA 和 ROCm 设备，提供统一的设备抽象

### 平台支持

Open MPI 支持广泛的平台：

- 操作系统：Linux、macOS、Cygwin、OpenBSD 等
- 架构：x86_64、ARM、PowerPC 等
- 运行时系统：Slurm、PBS、LSF、SGE 等

还有一些其他我不关心的东西没有在此处列出，比如I/O子系统。

## 如何开始

### 安装openmpi

你可以从源码编译安装openmpi，也可以使用包管理器安装。

这对一个稳定的发行版来说都很容易，比如Ubuntu、CentOS、Arch Linux等。

如果使用源码编译，你可以轻松的通过以下命令做到：

```
./autogen.pl && ./configure --prefix=/opt/openmpi && make -j64 && make install
```

请注意，如果通过tarball下载的源码而不是git repo, 则不需要执行autogen.pl命令。

### 编写MPI程序

先从简单的helloworld开始：

```
#include <mpi.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    char hostname[256];
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    gethostname(hostname, sizeof(hostname));
    printf("Hello from rank %d of %d on %s\n", rank, size, hostname);
    MPI_Finalize();
    return 0;
}
```

目前不需要理解他们的含义，我们在这里只是作为演示。

运行以下命令进行编译：

```
mpicc -o helloworld helloworld.c
```

### 运行程序

如果你只有一个机器，你可以直接运行：

```
mpirun -np 2 --mca btl self,sm helloworld
```

如果你有多个机器，太棒了，可以体验用多个机器并发任务了。

首先需要保证他们都安装了openmpi, 并且helloworld都在同一个目录下。

然后编写一个host file去澄清他们的ip地址：

```
192.168.1.101 slots=8
192.168.1.102 slots=8
```

slots表示每块板子能并行的核心数，通常是芯片的cpu核心数

现在开始运行：

```
mpirun -np 2 --hostfile hosts.txt /home/troy/helloworld
```

Enjoy :)
