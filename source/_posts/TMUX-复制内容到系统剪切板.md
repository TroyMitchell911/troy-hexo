title: '[TMUX]复制内容到系统剪切板'
date: '2024-10-05 19:12:12'
updated: '2024-10-05 19:12:15'
tags:
  - tmux
---
安装软件：

```bash
sudo apt install xclip  # 或者使用 xsel
```

在``～/.tmux.conf`或者`~/.tmux.conf.local`中添加如下内容：

```bash
bind-key -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "xclip -sel clip -i"
```

重新加载`tmux`配置：

```bash
tmux source-file ~/.tmux.conf
```