---
title: Zookeeper集群搭建
tags: ["Zookeeper", "集群", "高可用"]
categories: ["Zookeeper"]
---
本篇以**一主双从+一观察者**为例搭建Zookeeper集群，旨在介绍集群中的一些概念以及特性。
# 集群角色
## Leader
领导者，也可以叫做主节点。主节点可以处理读写请求，集群中只能有一个主节点。
## Follower
跟随者，也可以叫做从节点。从节点只处理读请求，同时作为Leader节点的候选节点。即如果Leader宕机，Follower节点会参与到新的Leader选举中，有可能成为新的Leader节点。
## Observer
观察者，只能处理读请求，并且不参与选举。通常是为了缓解整个集群的压力，提升集群的并发能力。

<!-- more -->

# 集群架构
![Zookeeper集群.png](Zookeeper集群.png)
# 集群搭建
## 创建节点唯一ID
在Zookeeper的安装目录下，创建如下data目录结构：
![节点唯一ID.png](节点唯一ID.png)
其中的myid文件（文件名固定，不可随意起名）存储代表各个节点的唯一ID，如下：
![节点唯一ID2.png](节点唯一ID2.png)
## 编辑配置文件
在conf目录下，复制zoo_sample.cfg四份，分别命名：zk1.cfg、zk2.cfg、zk3.cfg、zk4.cfg，并进行如下配置：
```bash
-- zk2、zk3、zk4
dataDir=/usr/local/apache-zookeeper-3.5.8-bin/data/zk1

-- 2182、2183、2184
clientPort=2181

server.1=192.168.175.175:2001:3001
server.2=192.168.175.175:2002:3002
server.3=192.168.175.175:2003:3003
server.4=192.168.175.175:2004:3004:observer
```
其中server.A=B:C:D:E，各自含义如下：
* A：节点唯一ID
* B：节点IP
* C：集群通讯端口
* D：集群选举端口
* E：集群角色，默认是participant，即参与者，也就是参与过半机制的角色。另一个是observer，观察者，不参与过半机制。

> 过半机制有两个场景，第一是选举，第二是事务请求过半提交。

## 集群启动
集群的启动没有什么特殊之处，就是依次启动各个节点。
![集群启动.png](集群启动.png)
然后查看一下集群状态：
![集群状态.png](集群状态.png)
可以看到：主节点为zk2，从节点为zk1、zk3，观察者如配置为zk4。

# 集群动态配置
Zookeeper 3.5.0 以前，Zookeeper集群角色要发生改变的话，只能通过停掉所有的Zookeeper服务，修改集群配置，重启服务来完成，这样集群服务将有一段不可用的状态，为了应对高可用需求，Zookeeper 3.5.0 提供了支持动态扩容/缩容的新特性。但是通过客户端API可以变更服务端集群状态是件很危险的事情，所以在zookeeper 3.5.3 版本要用动态配置，需要开启超级管理员身份验证模式 ACLs。如果是在一个安全的环境也可以通过配置系统参数`-Dzookeeper.skipACL=yes`来避免配置维护acl权限配置。下面我们就承接上文的集群架构，来看看如何进行集群的动态配置。

## 权限处理
可以通过**添加超级管理员**或者**跳过权限认证**

## 修改配置文件
修改所有集群节点的配置文件（zk1.cfg, zk2.cfg, zk3.cfg, zk4.cfg），如下：
* 移除clientPort配置
* 移除集群server.x配置
* 添加reconfigEnabled配置，设置为true开启动态配置
* 添加dynamicConfigFile配置，指定动态配置文件的位置

## 添加动态配置文件
约定俗成，文件以.dynamic结尾，如下：
![动态配置.png](动态配置.png)
文件内容如下：
```bash
server.1=192.168.175.175:2001:3001:participant;192.168.175.175:2181
server.2=192.168.175.175:2002:3002:participant;192.168.175.175:2182
server.3=192.168.175.175:2003:3003:participant;192.168.175.175:2183
server.4=192.168.175.175:2004:3004:observer;192.168.175.175:2184
```
> 在原来集群配置`server.A=B:C:D:E`的基础上加了一个F，变成了`server.A=B:C:D:E;F`，F代表服务ip:端口，**注意E和F以分号隔开**

## 客户端操作
依次启动所有节点，连接任意一台服务进行操作：
```bash
# 启动节点
./bin/zkServer.sh start conf/zk1.cfg
./bin/zkServer.sh start conf/zk2.cfg
./bin/zkServer.sh start conf/zk3.cfg
./bin/zkServer.sh start conf/zk4.cfg

# 查看节点状态
./bin/zkServer.sh status conf/zk1.cfg
./bin/zkServer.sh status conf/zk2.cfg
./bin/zkServer.sh status conf/zk3.cfg
./bin/zkServer.sh status conf/zk4.cfg

# 如果要修改集群状态，需要超级管理员登录（或者跳过授权）
addauth digest super:123456

# 移除id为3的机器
reconfig -remove 3

# 添加对应机器
reconfig -add server.3=192.168.175.175:2003:3003:participant;192.168.175.175:2183
```
可以看到该过程集群配置变化如下：
![集群动态配置.png](集群动态配置.png)
> 如果要变更或者添加新的服务，需要满足以下几点：
* 新服务必须是动态配置节点（配置文件要符合要求）
* 要将新服务添加到配置文件zk1.cfg.dynamic（注意并不是zk1.cfg.dynamic.xxx）中
* 新服务要处于启动状态
* 连接任意一个集群客户端，执行`reconfig -add`操作
* 保证服务列表中participant角色能够形成集群（过半机制）














