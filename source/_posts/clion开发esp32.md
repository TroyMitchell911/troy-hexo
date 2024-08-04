title: clion开发esp32
date: '2024-08-01 19:35:24'
updated: '2024-08-04 19:35:29'
tags:
  - mcu
categories:
  - ide
  - mcu
---
## README

本文基于文末的`ref`链接[1]搭建，这里作为一些补充记录。

Chip: esp32c3

System: Ubuntu 22.04

## 安装idf出现错误

在执行`.install.sh`时遇到如下错误：

```bash
./install.sh
Detecting the Python interpreter
Checking "python3" ...
Python 3.10.12
"python3" has been detected
Checking Python cref: nsa-esp-elf-gdb, riscv32-esp-elf-gdb, xtensa-esp-elf, riscv32-esp-elf, esp32ulp-elf, openocd-esp32, esp-rom-elfs
Skipping xtensa-esp-elf-gdb@14.2_20240403 (already installed)
Skipping riscv32-esp-elf-gdb@14.2_20240403 (already installed)
Skipping xtensa-esp-elf@esp-13.2.0_20240530 (already installed)
Skipping riscv32-esp-elf@esp-13.2.0_20240530 (already installed)
Skipping esp32ulp-elf@2.38_20240113 (already installed)
Skipping openocd-esp32@v0.12.0-esp32-20240318 (already installed)
Skipping esp-rom-elfs@20240305 (already installed)
Installing Python environment and packages
Python 3.10.12
pip 22.0.2 from /home/troy/.espressif/python_env/idf5.3_py3.10_env/lib/python3.10/site-packages/pip (python 3.10)
Upgrading pip and setuptools...
Requirement already satisfied: pip in /home/troy/.espressif/python_env/idf5.3_py3.10_env/lib/python3.10/site-packages (22.0.2)
Collecting pip
  Using cached pip-24.2-py3-none-any.whl (1.8 MB)
Requirement already satisfied: setuptools in /home/troy/.espressif/python_env/idf5.3_py3.10_env/lib/python3.10/site-packages (59.6.0)
Collecting setuptools
  Using cached setuptools-72.1.0-py3-none-any.whl (2.3 MB)
Installing collected packages: setuptools, pip
  Attempting uninstall: setuptools
    Found existing installation: setuptools 59.6.0
    Uninstalling setuptools-59.6.0:
      Successfully uninstalled setuptools-59.6.0
  Attempting uninstall: pip
    Found existing installation: pip 22.0.2
    Uninstalling pip-22.0.2:
      Successfully uninstalled pip-22.0.2
Successfully installed pip-24.2 setuptools-72.1.0
Skipping the download of /home/troy/.espressif/espidf.constraints.v5.3.txt because it was downloaded recently.
Installing Python packages
 Constraint file: /home/troy/.espressif/espidf.constraints.v5.3.txt
 Requirement files:
  - /home/troy/repo/esp-idf-v5.3/tools/requirements/requirements.core.txt
Usage: __main__.py [options]

ERROR: Invalid requirement: --2024-08-01 15:30:43--  https://dl.espressif.com/dl/esp-idf/espidf.constraints.v5.3.txt
__main__.py: error: no such option: --2024-08-01

Traceback (most recent call last):
  File "/home/troy/repo/esp-idf-v5.3/tools/idf_tools.py", line 3241, in <module>
    main(sys.argv[1:])
  File "/home/troy/repo/esp-idf-v5.3/tools/idf_tools.py", line 3233, in main
    action_func(args)
  File "/home/troy/repo/esp-idf-v5.3/tools/idf_tools.py", line 2680, in action_install_python_env
    subprocess.check_call(run_args, stdout=sys.stdout, stderr=sys.stderr, env=env_copy)
  File "/usr/lib/python3.10/subprocess.py", line 369, in check_call
    raise CalledProcessError(retcode, cmd)
subprocess.CalledProcessError: Command '['/home/troy/.espressif/python_env/idf5.3_py3.10_env/bin/python', '-m', 'pip', 'install', '--no-warn-script-location', '-r', '/home/troy/repo/esp-idf-v5.3/tools/requirements/requirements.core.txt', '--upgrade', '--constraint', '/home/troy/.espressif/espidf.constraints.v5.3.txt', '--extra-index-url', 'https://dl.espressif.com/pypi']' returned non-zero exit status 1.
``

手动下载：

```bash
wget https://dl.espressif.com/dl/esp-idf/espidf.constraints.v5.3.txt -O /home/troy/.espressif/espidf.constraints.v5.3.txt

--2024-08-01 15:43:03--  https://dl.espressif.com/dl/esp-idf/espidf.constraints.v5.3.txt
正在连接 127.0.0.1:7897... 已连接。
已发出 Proxy 请求，正在等待回应... 200 OK
长度： 2940 (2.9K) [text/plain]
正在保存至: ‘/home/troy/.espressif/espidf.constraints.v5.3.txt’

/home/troy/.espressif/espidf.constrai 100%[========================================================================>]   2.87K  --.-KB/s    用时 0s    

2024-08-01 15:43:03 (446 MB/s) - 已保存 ‘/home/troy/.espressif/espidf.constraints.v5.3.txt’ [2940/2940])
```

## 烧录程序后如何自动打开串口

在经过对配置文件`monitor`的尝试之后，确实以`115200`波特率打开了`/dev/ttyACM0`，但是`console`上显示的字符全是空白，不清楚为什么，所以为`flash`这个`configuration`添加一下可执行文件，记得勾选上在`输出控制台模拟终端`。

可执行文件内容：

```bash
#!/bin/bash

idf.py monitor
```

PS: 由于用到了`idf.py`，所以要执行`export.sh`，在`toolchain`里面的环境变量中可以添加`export.sh`.

![how to install export.sh](https://i.ibb.co/42jkNps/2024-08-04-19-28-11.png)

记得勾选上

## Ref

[1]: https://blog.csdn.net/m0_51719399/article/details/127389279
[2]: https://www.bilibili.com/read/cv15226500/
[3]: https://github.com/TroyMitchell911/esp32-example-clion
[4]: https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32c3/api-guides/jtag-debugging/index.html
