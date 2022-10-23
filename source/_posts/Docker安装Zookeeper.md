---
title: Docker安装Zookeeper
tags: ["Zookeeper", "Docker"]
categories: ["Docker"]
---

# 查看可用版本
访问 Zookeeper 镜像库地址：https://hub.docker.com/_/zookeeper/tags

<!-- more -->

![查询可用镜像.png](查询可用镜像.png)

# 拉取Zookeeper镜像
```bash
$ docker pull zookeeper:3.5.8
```

# 查看本地镜像
```bash
$ docker images
```
![查看本地镜像.png](查看本地镜像.png)

# 运行Zookeeper容器
```bash
$ docker run -itd --name zookeeper -p 2181:2181 zookeeper:3.5.8
```
* -p 2181:2181：映射容器服务的 2181 端口到宿主机的 2181 端口，外部主机可以直接通过 `宿主机ip:2181` 访问到 Zookeeper 的服务

# 验证启动成功
```bash
$ docker ps
```
![验证启动.png](验证启动.png)














