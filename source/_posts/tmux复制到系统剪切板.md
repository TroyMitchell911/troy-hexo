title: tmux复制到系统剪切板
date: '2024-11-30 12:14:39'
updated: '2024-11-30 12:14:40'
tags:
  - ubuntu
---
tmux可以使用这个配置：[oh my tmux](https://github.com/gpakosz/.tmux)

然后在`.tmux.conf.local`中将以下选项设置为true：

```bash
# -- clipboard -----------------------------------------------------------------

# in copy mode, copying selection also copies to the OS clipboard
#   - true
#   - false (default)
#   - disabled
# on Linux, this requires xsel, xclip or wl-copy
tmux_conf_copy_to_os_clipboard=true
```