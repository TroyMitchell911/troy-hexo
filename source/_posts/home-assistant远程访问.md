title: home assistant远程访问
date: '2024-08-24 16:55:09'
updated: '2024-08-26 10:36:46'
tags:
  - home-assistant
categories:
  - home-assistant
---
根据链接创建内网穿透：https://sspai.com/post/79278

此时打开域名会显示如下信息：

![bad request](https://developer.qcloudimg.com/http-save/yehe-10439143/0b96f1157a009111e282a8f2c97323b0.png)

查看`home assistant`的日志：

![log](https://developer.qcloudimg.com/http-save/yehe-10439143/28809a086eb9acbfd4382ab534929919.png)

此时可以看到一个`ip`，记录下来。

打开`homeassistant`的配置文件：

```bash
vim 
# 文件末尾添加以下内容
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - <log中的ip地址>
```

重启`ha`，问题解决

## Ref

https://cloud.tencent.com/developer/article/2260090
