title: Linux服务器配置ssh密钥登录
date: '2024-08-12 11:38:50'
updated: '2024-08-12 12:57:43'
tags:
  - linux
  - ubuntu
categories:
  - server
---
## What server does

```bash
sudo vim /etc/ssh/sshd_config
```

打开如下选项：

```bash
PubkeyAuthentication yes
```

使用如下命令生成密钥：

```bash
ssh-keygen

cd .ssh

cat id_rsa.pub >> authorized_keys
```

重启`sshd`服务：

```bash
sudo systemctl restart ssh
```

## What client does

复制`.ssh`目录下的`id_rsa`文件到`~/.ssh`目录下并重命名一个名字。

```bash
chmod 600 ./<your id_rsa name> 
```

编辑`hostname`

```bash
vim ~/.ssh/config
```

将以下内容添加进入：

```bash
Host <your server name>
HostName <your server IP>
TCPKeepAlive yes
ServerAliveInterval 15
User <your user name of server>
IdentityFile <your id_rsa file path>
Port <your server port>
```

尝试连接：

```bash
ssh <your server name>
```