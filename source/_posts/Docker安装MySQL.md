---
title: Docker安装MySQL
tags: ["MySQL", "Docker", "容器"]
categories: ["Docker"]
---

# 查看可用版本
访问 MySQL 镜像库地址：https://hub.docker.com/_/mysql?tab=tags

<!-- more -->

![查询可用镜像.png](查询可用镜像.png)

# 拉取MySQL镜像
```bash
$ docker pull mysql:8.0.31
```
![拉取MySQL镜像.png](拉取MySQL镜像.png)

# 查看本地镜像
```bash
$ docker images
```
![查看本地镜像.png](查看本地镜像.png)

# 运行MySQL容器
```bash
$ docker run -itd --name mysql8 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:8.0.31
```
* -p 3306:3306：映射容器服务的 3306 端口到宿主机的 3306 端口，外部主机可以直接通过 `宿主机ip:3306` 访问到 MySQL 的服务
* MYSQL_ROOT_PASSWORD=123456：设置MySQL服务root用户的密码

# 验证启动成功
```bash
$ docker ps
```
![验证启动.png](验证启动.png)














