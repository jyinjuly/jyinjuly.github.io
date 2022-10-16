---
title: CentOS安装Docker
tags: ["CentOS", "Docker", "容器"]
categories: ["Docker"]
---

# 卸载旧版本
较旧的 Docker 版本称为 docker 或 docker-engine 。如果已安装这些程序，请卸载它们以及相关的依赖项。
```bash
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```
# 安装 Docker
## 安装前置依赖
```bash
$ sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```
## 设置 Docker 仓库
### 官方源
```bash
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```
### 阿里云源
```bash
$ sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
### 清华大学源
```bash
$ sudo yum-config-manager \
    --add-repo \
    https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo
```
## 执行 Docker 安装
### 最新版本
```bash
$ sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
### 指定版本
要安装特定版本的 Docker，请在存储库中列出可用版本，然后选择并安装
```bash
yum list docker-ce --showduplicates | sort -r
```
![Docker版本选择.png](Docker版本选择.png)
通过其完整的软件包名称安装特定版本，该软件包名称是软件包名称（docker-ce）加上版本字符串（第二列），从第一个冒号（:）一直到第一个连字符，并用连字符（-）分隔。例如：docker-ce-18.09.1。
```bash
$ sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
```
# 启动 Docker
```bash
$ sudo systemctl start docker
```
# 验证启动
通过运行 hello-world 镜像来验证是否启动成功（首次运行可能需要先拉取镜像）
```bash
$ sudo docker run hello-world
```
![Docker启动验证.png](Docker启动验证.png)
# 关闭 Docker
```bash
$ sudo systemctl stop docker
```
![Docker自动唤醒机制.png](Docker自动唤醒机制.png)
# 卸载 Docker
删除安装包：
```bash
yum remove docker-ce
```
删除镜像、容器、配置文件等内容：
```bash
rm -rf /var/lib/docker
```















