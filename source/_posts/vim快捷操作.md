title: vim快捷操作
date: '2024-07-02 22:42:21'
tags:
  - vim
---
## 快速定位

- A: 跳到行尾并开启编辑模式
- 0: 跳到行首
- $: 跳到行尾
- G: 跳到文件尾
- gg: 跳到文件首
- nG: 跳到第n行
    - e.g. 50G: 跳到第50行
- nj: 向下跳n行
    - e.g. 3j: 向下跳3行
- nk: 向上跳n行
    - e.g. 3k: 向上跳3行
- nw: 向后跳n个单词
    - e.g. 3w: 向后跳3个单词
 - nb: 向前跳n个单词
    - e.g. 3b: 向前跳3个单词
- /: 搜索
    - e.g. /123: 搜索123文本
    - 按n和N向下和向上

### pair

- %: 在pair中来回跳
    - e.g. (123)  在（和）之间来回跳
- ci + left-pair: 删除pair中的内容并且开启编辑模式
    - e.g. ci+(: 删除（）中的内容并且开启编辑模式
- di + left-pair: 删除pair中的内容
    - e.g. di+(: 删除（）中的内容
- yi + left-pair: 复制pair中的内容
    - e.g. yi+(: 复制（）中的内容
- vi + left-pair: 选中pair中的内容
    - e.g. vi+(: 选中（）中的内容
- va + left-pair: 删除pair中的内容包括pair
    - e.g. va+(: 删除（）中的内容包括（）