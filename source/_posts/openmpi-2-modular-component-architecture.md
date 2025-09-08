---
title: openmpi[2] 模块化组件架构
date: 2025-09-08 15:49:21
tags:
  - openmpi
categories:
  - openmpi
---

Open MPI 是一个高度可定制的系统；它可以通过配置文件、命令行参数和环境变量进行配置。Open MPI 配置系统的主要功能是通过模块化组件架构 (MCA) 实现的。

## 一些术语

模块化组件架构 (MCA) 是 Open MPI 大部分功能的支柱。它由一系列项目 、 框架 、 组件和模块在运行时组装以创建 MPI 实现。

MCA 参数 （也称为 MCA 变量 ）用于定制 Open MPI 在运行时的行为。

### Project

Project本质上是 Open MPI 代码库中最高抽象层的划分。

但这里的Project可不是说的项目，而是指的Open MPI 代码库中主要的、顶层的代码部分

### Framework

MCA 框架在运行时管理零个或多个组件，并针对特定任务（例如，提供 MPI 集合操作功能）。虽然每个 MCA 框架仅支持一种类型的组件，但它可以支持多个该类型的组件。

比如btl framework仅支持btl类型的组件，但是它可以支持多个btl组件，比如openib、tcp、usnic等。

用户可能想自定义或者更改的framework如下：

- btl: Byte Transport Layer (字节传输层); 这些组件专门用作 PML(点对点) 组件的底层传输。
- coll: 集合操作层 (Collection Operations Layer)；这些组件专门用于实现 MPI 集合操作。
- io: MPI I/O
- mtl: MPI 匹配传输层 (MPI Matching Transport Layer)；这些组件专门用作 cm PML 组件的底层传输。
- pml: 点对点消息传递层 (Point-to-point Messaging Layer) 。这些组件用于实现 MPI 点对点消息传递功能。

### Components

MCA 组件是框架正式接口的实现。它是一个独立的代码集合，可以捆绑成插件，并在运行时或编译时插入到 Open MPI 代码库中。

Open MPI 的Components概念的同义词是“plugin”或“add-on”。

### Module

Module是Component的一个实例（在 C++ 意义上 词“实例”；Component类似于 C++ 类）。例如，如果运行 Open MPI 应用程序的节点有两个以太网 NIC，Open MPI 应用程序将包含一个 TCP MPI 点对点 Component ，而是两个 TCP 点对点Modules 。

### MCA 参数

MCA 参数 （有时称为 MCA 变量 ）是 Open MPI 运行时调优的基本单位。它们是简单的“键 = 值”对，在 Open MPI 中广泛使用。开发人员使用的一般经验规则是：

- 不要将重要值设为常数，而应将其设为 MCA 参数。
- 如果一个任务可以通过多种用户可辨别的方式实现，则尽可能多地实现，并使用 MCA 参数在运行时在它们之间进行选择。

## 设置MCA参数（变量）

MCA参数可以通过以下几种方式进行设置，按优先级列出：

1. 命令行
2. 环境变量
3. mca参数文件
4. 配置文件

### 命令行

优先级最高的方法是在命令行中设置 MCA 参数。例如：

```
shell$ mpirun --mca mpi_show_handle_leaks 1 -np 4 a.out
```

这会在使用四个进程运行 a.out 之前，将 MCA 参数 mpi_show_handle_leaks 的值设置为 1。通常，命令行中使用的格式为 --mca <param_name> <value> 。

设置包含空格的值时，需要使用引号，以确保 Shell 能够将多个标记理解为一个值。例如：

```
shell$ mpirun --mca param "value with multiple words" ...
```

### 环境变量

### mca参数文件

可以使用简单的文本文件来设置特定应用程序的 MCA 参数值。

mpirun --tune CLI 选项允许用户在单个文件中指定 MCA 参数和环境变量。

调整参数文件中设置的 MCA 参数将覆盖任何 MCA 全局参数文件中提供的参数（例如， $HOME/.openmpi/mca-params.conf )，但不是命令行或环境参数。

假设有一个已调整的参数文件名为 foo.conf ，它与应用程序 a.out 放在同一目录中。用户通常会以如下方式运行该应用程序：

```
shell$ mpirun -np 2 a.out
```

要使用 foo.conf 调整参数文件，此命令行更改为：

```
shell$ mpirun -np 2 --tune foo.conf a.out
```

如果要使用多个文件，则可以将已调优的参数文件组合在一起。如果还有一个名为 bar.conf 的已调优参数文件，则可以按如下方式将其添加到命令行：

```
shell$ mpirun -np 2 --tune foo.conf,bar.conf a.out
```

已调整文件的内容由一行或多行组成，每行包含零个或多个 -x 和 –mca 选项。不允许使用注释。例如，以下已调整文件：

```
-x envvar1=value1 -mca param1 value1 -x envvar2
-mca param2 value2
-x envvar3
```

相当于:

```
shell$ mpirun \
    -x envvar1=value1 -mca param1 value1 -x envvar2 \
    -mca param2 value2
    -x envvar3 \
    ...rest of mpirun command line...
```
