title: lazyvim复制到剪切板
date: '2024-11-30 12:07:41'
updated: '2024-11-30 12:07:43'
tags:
  - ubuntu
---
在neovim(10.0.2)中，其实已经默认开启了yank插件，也就是复制的内容会自动传入剪切板。

那么为什么还会有这篇文章呢？因为lazyvim默认有这样一个配置：

```bash
opt.clipboard = vim.env.SSH_TTY and "" or "unnamedplus" -- Sync with system clipboard
```

这句的意思查看你的shell是否是tty类型，如果是tty那么就不会进入系统剪切板。恰好ssh就是tty类型，所以ssh连接的shell在打开nvim复制的内容是不会进入到系统剪切板的。

进入系统剪切板的作用：[here](https://blog.troy-y.org/2024/11/30/ssh%E5%8F%8C%E5%90%91%E5%A4%8D%E5%88%B6/)

所以要在`~/.config/nvim/lua/config/option.lua`中覆盖这条默认设置：

```bash
opt.clipboard = "unnamedplus"
```

这样即便tty也能够进入系统剪切板了。