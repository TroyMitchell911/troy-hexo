title: OpenHarmony标准系统认证
date: '2024-09-04 10:12:22'
updated: '2024-09-04 10:12:24'
tags:
  - openHarmony
---
## 环境配置

**该部分在Windows上完成**

确保`python`版本为`3.7`以上，`3.7.8`是推荐的，但不是绝对的：

```
python --version
Python 3.7.8
```

安装包：

```
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple setuptools
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple pyserial
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple rsa
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple python-dateutil
```

## Acts应用兼容性测试

在[这里](https://www.openharmony.cn/certification/document/xts/)选择`OH`对应的版本的套件和资源文件。

需要注意的一点就是，`Acts`套件如果是`arm32`可以直接下载，但是其他的需要在`OH`源代码目录进行编译。

由于这里是arm64, 所以要编译一下`Acts`套件。

进入`OH`的源码根目录后：

```bash
$ cd test/xts/acts
$ ./build.sh product_name=rk3568 system_size=standard
```

编译完成后生成目录：

    - 测试用例输出目录: out/rk3568/suites/acts/testcases
    - 测试框架&用例整体输出目录: out/rk3568/suites/acts


复制`out/rk3568/suites/acts`到主机：

```bash
❯ cp -r acts ~/VirtualBox\ VMsWin10/share/oh
```

解压下载的资源文件压缩包到`acts`文件夹里面。

完成之后目录如下：

```bash
❯ tree -L 1
.
├── config
├── resource
├── run.bat
├── run.sh
├── testcases
└── tools

4 directories, 2 files
```

## Ref

https://www.openharmony.cn/certification/document/guid