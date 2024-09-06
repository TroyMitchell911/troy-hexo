title: docker上传至hub
date: '2024-09-06 21:11:43'
updated: '2024-09-06 21:11:46'
tags:
  - docker
---
## 将已有容器提交为镜像


如果你当前有的是一个正在运行的` Docker `容器，而不是镜像，你可以将这个容器保存为镜像，然后再上传到 `Docker Hub`。

可以使用` docker commit `命令，将当前容器保存为一个新的 Docker 镜像：

```bash
❯ docker commit <container-id> <new-image-name>
```

如果你的容器 ID 是 `abc123`，并且你想把它保存为名为 `my-app-image` 的镜像：

```bash
❯ docker commit abc123 my-app-image
```

**如果你需要附加信息，可以使用-m选项添加你要提交的信息**

使用`docker images`可以查看生成的镜像。

## 标记镜像

现在已经有了一个镜像，即便没有，是容器的话，经过上一步骤也应该有了镜像，现在需要给镜像打标签标记版本：

```bash
❯ docker tag <new-image-name> <hub-username>/<repository-name>:<tag>
```

## 推送镜像

现在可以将标记的镜像推送到`docker hub`了:

```bash
❯ docker push <hub-username>/<repository-name>:<tag>
```

## 多个标记

在 `Docker Hub `中，你可以为同一个镜像创建多个标签（`tags`），例如 `latest`、`v1.1`、`v1.2 `等，这样可以标识不同的版本，同时保持 `latest `作为最新版本的标识。

```bash
❯ docker tag <new-image-name> <hub-username>/<repository-name>:latest
❯ docker tag <new-image-name> <hub-username>/<repository-name>:v1.1
❯ docker push <hub-username>/<repository-name>:latest
❯ docker push myusername/<repository-name>:v1.1
```