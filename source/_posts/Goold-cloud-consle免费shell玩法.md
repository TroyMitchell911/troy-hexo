title: Google cloud consle免费shell玩法
date: '2024-10-31 23:08:01'
updated: '2024-10-31 23:08:02'
tags:
  - network
categories:
  - network
---
## Prep

Google cloud consle我并不清楚是个什么东西，由[percycle](https://github.com/per1cycle)提供给我。直到现在，我只知道我能通过它创建12个shell配额。

此`shell`运行在`docker`中，并没有提供`公网IP`，这意味着可玩性大大降低，不能当作`server`使用。

目前已知的玩法：
- Tailscale exit node

Link here: https://console.cloud.google.com/

**PS:本文假设您已经注册了tailscale并且了解tailscale与tailscale exit node是什么**

## Tailscale

首先创建此文件以摆脱烦人的警告：

```bash
$ mkdir -p ~/.cloudshell/
$ touch ~/.cloudshell/no-apt-get-warning
```

使用如下命令安装tailscale:

```bash
$ curl -fsSL https://tailscale.com/install.sh | sh
```

此时收到调用`tailscale up`进行登陆，但这里不要直接使用该命令，因为我们需要作为`exit node`使用，所以要通过如下命令:

```bash
$ sudo tailscale up --advertise-exit-node
```

执行该命令后应该会收到一条提示，提示`tailscaled`并没有运行，这可能是因为`tailscaled`默认使用`systemd`启动，而这个容器并没有提供`systemd`。

可以直接调用`tailscaled`或者使用`service`启动：

```bash
$ sudo tailscaled
# $ sudo service tailscaled start
```

重新运行up命令，此时应该收到一条提示登录的消息，在电脑上打开该链接进行登录。

登录之后需要在`tailscale`控制台打开该`exit node`：

[![2024-10-31-23-03-38.png](https://i.postimg.cc/7ZCT6Fr3/2024-10-31-23-03-38.png)](https://postimg.cc/k2m5Yh4G)

[![2024-10-31-23-04-11.png](https://i.postimg.cc/gjkxP5K6/2024-10-31-23-04-11.png)](https://postimg.cc/gXQzVgQY)

此时就应该将流量转发到`exit node`了，这个容器应该会根据你开的梯的IP地址所开启，我开的是台湾的梯，容器就是台湾的。这一点可以通过`ip138`确认是否流量转发成功:

[![2024-10-31-22-17-23.png](https://i.postimg.cc/2jc6Z0w2/2024-10-31-22-17-23.png)](https://postimg.cc/PNDTGz0w)

此外，该容器没有外网ip，可以使用`tailscale`提供的局域网ip进行连接，不过关闭了密码连接，我们也不知道密码。可以使用`ssh key`进行连接，这点是很通常的操作，我留给你去自己完成。

提供成功`ssh`截图：

[![2024-10-31-22-39-31.png](https://i.postimg.cc/CLCw4JC3/2024-10-31-22-39-31.png)](https://postimg.cc/kB4rJQps)

## Ref

https://tailscale.com/download
https://tailscale.com/kb/1103/exit-nodes?tab=linux

