title: ssh双向复制
date: '2024-11-30 11:45:14'
updated: '2024-11-30 11:45:16'
tags:
  - ubuntu
---
在主机A通过ssh连接到主机B时，我们在A内复制的文本可以通过Ctrl+Shift+v粘贴到B，但是B内复制的却不能与A共享。

可能这里会造成一个疑惑：明明我复制ssh的shell信息也可以粘贴出来。其实这里复制复制的是终端上的字，并不是ssh的shell内复制，还是相当于本机复制。

这一点可以通过xsel工具做验证：

```bash
# A
❯ xsel --clipboard --output

Linux troy-server 6.8.0-49-generic #49~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Wed Nov  6 17:42:15 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
\ No newline at end of selection
# B
❯ xsel --clipboard --output    
```

可以看到虽然A复制的B的终端输出信息，但是却没有进入B的剪切板，而是进入A的剪切板，说明复制的是终端上的字，并不是ssh的shell内复制。

如果这里做一个假设，在B的shell打开一个vim进行复制（已经配置好vim复制内容进入剪切板），此时B复制的内容A还能粘贴出来吗？答案是肯定不行的，因为进入的是B的剪切板而不是A的。

tmux也是同理，tmux中复制进入的是tmux的bufer，我们也可以配置成进入系统剪切板。但无论如何，开在B上面的tmux或vim进入的都是B的系统剪切板。所以这里要做的工作就是将A和B的系统剪切板共享。

## X11转发配置

首先确保B上面安装了X11软件包(如果是server版本可能并没有安装)：

```bash
❯ sudo apt install xorg xauth x11-apps xterm
```

在B上面修改`/etc/ssh/sshd_config`以支持X11转发：

```bash
X11Forwarding yes
```

重启sshd服务：

```bash
❯sudo systemctl restart sshd
```

在A上面通过ssh连接启用X11转发有两种方式：

- 方式1: `ssh -Y <username>@<ip-addr>`
- 方式2: 添加`ForwardX11 yes`到config

此时从A进入到B的shell中后，就可以运行如下命令测试x11是否转发：

```bash
$ xclock
```

## 相关阅读

在服务器配合这两个文档基本可以横着走：

[tmux复制到剪切板](https://blog.505218.xyz/2024/11/30/tmux%E5%A4%8D%E5%88%B6%E5%88%B0%E7%B3%BB%E7%BB%9F%E5%89%AA%E5%88%87%E6%9D%BF/)
[lazyvim复制到剪切板](https://blog.505218.xyz/2024/11/30/lazyvim%E5%A4%8D%E5%88%B6%E5%88%B0%E5%89%AA%E5%88%87%E6%9D%BF/)