title: 使用vim阅读代码
date: '2024-07-02 19:30:17'
tags:
  - vim
  - ubuntu
categories:
  - ide
  - ubuntu
---
## 环境
- Ubuntu22.04
- VIM - Vi IMproved 8.2 (2019 Dec 12, 编译于 May 03 2024 02:37:51)

## 引言
在`Linux`环境下，`Source Insight`只能在Wine环境下运行，显得并没有那么好用，于是便有了本文，使用`Vim+Ctags+Cscope`来进行阅读代码。

## 安装

### 安装Ctags

打开终端，运行以下命令安装 `ctags`：

```bash
sudo apt install exuberant-ctags
```

如果你希望使用 `universal-ctags`（这是一个更新和维护更积极的分支），则可以安装它：

```bash
sudo apt install universal-ctags
```

### 安装Cscope

```bash
sudo apt install cscope
```

## 配置与使用

### 配置Ctags

在项目根目录执行如下命令生成`tags`文件：

```bash
ctags -R .  
```

可以发现在项目根目录下多了如下文件：

```bash
❯ ls tags
tags
```

为了能够在项目中使用该文件作为`tag`索引，则在`~/.vimrc`中增加如下配置，这个配置的目的是为了能够让`vim`在项目的任意目录中都能够找到`tags`的配置：

```bash
set tags=./tags;/
```

### 使用Ctags

- `ctrl+t `:在跳转之后，按下ctrl+t,vim就会返回之前的跳转位置
- `ctrl+] `:将光标移动到函数或变量名上，按下ctrl+],vim会自动跳转到该函数或者变量的定义处
- `ctrl + w + ctrl + ]`: 将光标移动到函数或变量名上，按下快捷键,vim会自动跳转到该函数或者变量的定义处并垂直拆分标签页
- `ctrl +w +w`: 在刚才垂直拆分的标签页中来回跳转光标
- `ctrl + w + c`: 关闭一个垂直拆分的标签页
- `:tags `: 输入:tags, vim会显示所有可跳转的列表

### 配置Cscope


在项目根目录执行如下命令生成`tags`文件：

```bash
cscope -Rbq
```

可以发现在项目根目录下多了如下文件：

```bash
❯ ls cscope*
cscope.in.out  cscope.out  cscope.po.out
```


为了能够在项目中使用这些文件作为索引，则在下载`cscope`的官方配置文件：

```bash
wget -O ~/.cscope_maps.vim https://cscope.sourceforge.net/cscope_maps.vim
```

并且在`.vimrc`中增加配置：

```bash
source ~/.cscope_maps.vim
```

使用方法：在`vim`中光标放到一个`symbol`上，随后使用`vim`中配置的快捷键（details: ~/.cscope_maps.vim）。
e.g.: `ctrl + \ + c`: 搜索光标处的`symbol`在工程中被谁调用了。

## 创建脚本

为了每次创建`cscope`和`ctag`索引的方便快捷性，可以在`/usr/local/bin`目录下创建一个名为`ctcs`的脚本，脚本内容如下：

```bash
#!/bin/bash
#
# ctcs

star()
{
    ctags -R *   
    if [ $? -eq 0 ]; then
        echo "ctags successfully!"
    fi  
    cscope -qbR 
    if [ $? -eq 0 ]; then
        echo "cscope successfully!"
    fi  
}

del()
{
    if [ -f tags -o -f cscope.out -o -f cscope.po.out -o -f cscope.in.out ]; then
        rm -f tags  && echo "clean tags ok!"
        rm -f cscope.* && echo "clean cscope.* files ok!"
    fi  
}

case "$1" in
    -r) 
        del 
        star
        ;;  
    -d) 
        del 
        ;;  
    *)  
        echo "usage : ctcs -r|-d"   
        exit 1
esac

exit 0
```

随后使用如下命令赋予执行权限：

```bash
chmod +x /usr/local/bin/ctcs
```

## Ref
https://cscope.sourceforge.net/cscope_maps.vim
https://stackoverflow.com/questions/563616/vim-and-ctags-tips-and-tricks
https://bigeast.github.io/Source_Code_Tree_Navigation_in_Vim.html
https://segmentfault.com/a/1190000044591252
https://blog.csdn.net/zg_hover/article/details/78917921