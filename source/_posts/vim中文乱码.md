title: vim中文乱码
date: '2024-08-07 14:39:40'
updated: '2024-08-07 14:39:42'
tags:
  - vim
---
```bash
vim ~/.vimrc

# 将以下内容加入：
set termencoding=utf-8
set encoding=utf8
set fileencodings=utf8,ucs-bom,gbk,cp936,gb2312,gb18030
```
