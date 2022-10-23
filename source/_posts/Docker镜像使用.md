---
title: Docker镜像使用
tags: ["Docker", "镜像"]
categories: ["Docker"]
---

# 查看本地镜像
```bash
$ docker images
```
![查看本地镜像.png](查看本地镜像.png)
* REPOSITORY：表示镜像的仓库源
* TAG：镜像的标签
* IMAGE ID：镜像ID
* CREATED：镜像创建时间
* SIZE：镜像大小
<!-- more -->

同一仓库源可以有多个 TAG，代表这个仓库源的不同个版本，如 ubuntu 仓库源里，有 15.10、14.04 等多个不同的版本，我们使用 REPOSITORY:TAG 来定义不同的镜像。所以，我们如果要使用版本为15.10的ubuntu系统镜像来运行容器时，命令如下：
```bash
$ docker run -t -i ubuntu:15.10 /bin/bash
```
* -i: 交互式操作。
* -t: 终端。
* ubuntu:15.10: 这是指用 ubuntu 15.10 版本镜像为基础来启动容器。
* /bin/bash：放在镜像名后的是命令，这里我们希望有个交互式 Shell，因此用的是 /bin/bash。

**如果你不指定一个镜像的版本标签，例如你只使用 ubuntu，docker 将默认使用 ubuntu:latest 镜像（而如果你本地没有最新的镜像，则会自动拉取）。**

# 查找远程镜像
```bash
$ docker search mysql
```
![查找镜像.png](查找镜像.png)
* NAME: 镜像仓库源的名称
* DESCRIPTION: 镜像的描述
* OFFICIAL: 是否 docker 官方发布
* stars: 类似 Github 里面的 star，表示点赞、喜欢的意思。
* AUTOMATED: 自动构建。

# 拉取远程镜像
```bash
$ docker pull mysql:8.0.31
```

# 删除本地镜像
```bash
$ docker rmi mysql:8.0.31s
```










