---
title: Zookeeper单机版安装
tags: ["Zookeeper", "CentOS7", "客户端工具"]
categories: ["Zookeeper"]
---
# 安装Zookeeper
## 安装java环境
```bash
# java version "1.8.0_271"
java -version
```
## 下载并解压Zookeeper
```base
# 下载安装包
wget https://mirror.bit.edu.cn/apache/zookeeper/zookeeper‐3.5.8/apache‐zookeeper‐3.5.8‐bin.tar.gz
# 解压安装包
tar ‐zxvf apache‐zookeeper‐3.5.8‐bin.tar.gz
```
<!-- more -->
## 备份配置文件（非必须）
```bash
# 进入配置文件目录
cd apache-zookeeper-3.5.8-bin/conf/
# 备份配置文件
cp zoo_sample.cfg zoo.cfg
```
## 启动Zookeeper
```bash
# 进入脚本文件目录
cd apache-zookeeper-3.5.8-bin/bin/
# 启动Zookeeper
./zkServer.sh start ../conf/zoo.cfg
# 检测是否启动
ps -ef | grep zookeeper
```
# 客户端连接
## 终端方式连接
```bash
# 进入脚本文件目录
cd apache-zookeeper-3.5.8-bin/bin/
# 连接zookeeper
./zkCli.sh -server 192.168.175.175:2181
# 查看所有节点
ls -R /
```
## 可视化工具（ZooInspector）连接
* 下载工具：https://issues.apache.org/jira/secure/attachment/12436620/ZooInspector.zip
* 解压，进入目录ZooInspector\build
* 创建zkClient.bat脚本文件，内容如下：
```bat
java -jar D:\ZooInspector\build\zookeeper-dev-ZooInspector.jar
```
* 右键zkClient.bat文件，创建桌面快捷方式，然后修改一个自己喜欢的图标
* 双击快捷方式，打开可视化工具——ZooInspector，输入连接信息
![ZooInspector1.png](ZooInspector1.png)
* 点击OK，连接成功
![ZooInspector2.png](ZooInspector2.png)






















