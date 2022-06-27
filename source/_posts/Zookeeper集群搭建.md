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




















