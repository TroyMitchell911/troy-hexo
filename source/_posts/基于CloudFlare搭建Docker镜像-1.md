author: Troy Mitchell
title: 基于CloudFlare搭建Docker镜像
---
## 准备
- 一个域名
- 一个CloudFlare账号

## 域名修改DNS

PS:如果该站点已在CloudFlare中存在，请忽略该步骤。

首先在CloudFlare中添加一个站点，域名就写你自己的域名：

![upload successful](/images/pasted-0.png)

添加好之后在下面我们可以看到需要修改的DNS为：

![upload successful](/images/pasted-1.png)

这里我是使用的腾讯云的域名，所以以腾讯这里为例，修改好后如下图：

![upload successful](/images/pasted-2.png)

## 使用github部署

首先fork该仓库：https://github.com/ciiiii/cloudflare-docker-proxy

将`src/index.js`、`wrangler.toml`的`libcuda.so`替换成你的域名，以`src/index.js`为例：
```bash
vim src/index.js

:%s/libcuda.so/your-url/g

```
修改完成后push（如果在网页ui操作忽略该步骤）。

随后点击Deploy with works即可开始部署。

PS:
如果Delpoy with works有问题，请修改readme中该按钮的超链接为你自己的仓库地质

## Deploy with works

Account id就是https://dash.cloudflare.com/***中的***那一部分
Token需要自己在该网址进行创建：https://dash.cloudflare.com/profile/api-tokens
token模板就选择`编辑 Cloudflare Workers`即可。


