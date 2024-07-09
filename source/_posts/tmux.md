title: tmux
date: '2024-07-09 15:20:04'
updated: '2024-07-09 15:20:07'
tags:
  - linux
---
## 会话管理

### 进入

- `tmux`命令可以直接进入一个`session`

- 创建一个名为`session-name`的会话：

```bash
$ tmux new -s <session-name>
```

- 进入一个已经存在的会话
```bash
# 使用会话编号
$ tmux attach -t 0

# 使用会话名称
$ tmux attach -t <session-name>
```

### 退出

- `ctrl + d`可以直接退出
- `ctrl + b`后按`d`可以后台运行该会话，使用`attach`进入

### 其他

- `tmux ls` 或`ctrl +b s`列出所有会话
- `tmux kill-session -t`销毁某个会话
- `tmux rename-session`或`ctrl+b $`重命名某个会话


## 窗格管理

```bash
Ctrl+b %：划分左右两个窗格。
Ctrl+b "：划分上下两个窗格。
Ctrl+b <arrow key>：光标切换到其他窗格。<arrow key>是指向要切换到的窗格的方向键，比如切换到下方窗格，就按方向键↓。
Ctrl+b ;：光标切换到上一个窗格。
Ctrl+b o：光标切换到下一个窗格。
Ctrl+b {：当前窗格与上一个窗格交换位置。
Ctrl+b }：当前窗格与下一个窗格交换位置。
Ctrl+b Ctrl+o：所有窗格向前移动一个位置，第一个窗格变成最后一个窗格。
Ctrl+b Alt+o：所有窗格向后移动一个位置，最后一个窗格变成第一个窗格。
Ctrl+b x：关闭当前窗格。
Ctrl+b !：将当前窗格拆分为一个独立窗口。
Ctrl+b z：当前窗格全屏显示，再使用一次会变回原来大小。
Ctrl+b Ctrl+<arrow key>：按箭头方向调整窗格大小。
Ctrl+b q：显示窗格编号。
```

### 开启鼠标控制窗格

```bash
$ echo 'set -g mouse on'>> ~/.tmux.conf
$ tmux source-file ~/.tmux.conf
```

## 窗口管理

```bash
Ctrl+b c：创建一个新窗口，状态栏会显示多个窗口的信息。
Ctrl+b p：切换到上一个窗口（按照状态栏上的顺序）。
Ctrl+b n：切换到下一个窗口。
Ctrl+b <number>：切换到指定编号的窗口，其中的<number>是状态栏上的窗口编号。
Ctrl+b w：从列表中选择窗口。
Ctrl+b ,：窗口重命名。
```

