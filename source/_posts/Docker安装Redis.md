---
title: Docker安装Redis
tags: ["Redis", "Docker", "容器"]
categories: ["Docker"]
---

# 查看可用版本
访问 Redis 镜像库地址：https://hub.docker.com/_/redis?tab=tags

<!-- more -->

![查询Redis镜像.png](查询Redis镜像.png)

# 拉取Redis镜像
```bash
$ docker pull redis:5.0.14
```

# 查看本地镜像
```bash
$ docker images
```

# 运行MySQL容器
```bash
$ docker run -itd --name redis5 -p 6379:6379 redis:5.0.14
```

# 验证启动成功
```bash
$ docker ps
```














