---
title: Docker容器使用
tags: ["Docker", "容器"]
categories: ["Docker"]
---

# 获取镜像
```bash
$ docker pull mysql:8.0.31
```

# 启动容器
```bash
$ docker run --name mysql8 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:8.0.31
```
<!-- more -->

# 查询运行容器
```bash
$ docker ps
```
![查询运行容器.png](查询运行容器.png)
* CONTAINER ID: 容器id
* IMAGE: 镜像
* NAMES: 容器名称

# 停止容器
```bash
$ docker stop e60de6d38a35
```

# 查询所有容器
```bash
$ docker ps -a
```
![查询所有容器.png](查询所有容器.png)

# 启动停止容器
```bash
$ docker start e60de6d38a35 
```

# 重启运行容器
```bash
$ docker restart e60de6d38a35 
```

# 进入容器
```bash
$ docker exec -it e60de6d38a35 /bin/bash
```
> 使用exec进入容器，如果从这个容器退出，容器不会停止；也可以使用`docker attach e60de6d38a35`进入容器，但是如果从这个容器退出，会导致容器的停止。因此推荐使用exec。

# 删除容器
```bash
$ docker rm -f e60de6d38a35
```