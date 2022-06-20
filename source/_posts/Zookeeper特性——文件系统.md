---
title: Zookeeper特性——文件系统
tags: ["Zookeeper", "文件系统"]
categories: ["Zookeeper"]
---
Zookeeper有两大特性——文件系统和监听机制，本篇介绍文件系统。
# 文件系统简介
Zookeeper维护一个类似文件系统的数据结构，如下图：
![Znode.png](Znode.png)
<!-- more -->
其中每个目录项都被称为**Znode（节点）**，节点之下还可以增加子节点。每个节点拥有唯一路径path，并且均以根路径/开头，节点还可以保存数据。
# 节点Znode类型
## 持久化节点
PERSISTENT（默认类型节点），客户端与Zookeeper断开连接后，节点依旧存在。只要不手动删除，这类节点将永远存在。
## 持久化序号节点
PERSISTENT_SEQUENTIAL，与持久化节点一样，只是Zookeeper会自动给节点名称进行顺序编号。
> create -s
## 临时节点
EPHEMERAL­，客户端与Zookeeper断开连接后，节点会被删除。
> create -e
## 临时序号节点
EPHEMERAL_SEQUENTIAL，与临时节点一样，只是Zookeeper会自动给节点名称进行顺序编号。
> create -s -e
## 容器节点
Container，3.5.3版本新增。容器节点的表现形式和持久化节点是一样的，区别在于Zookeeper服务端启动后，会有一个单独的线程去扫描所有的容器节点，当发现容器节点的子节点数量为0时，会自动删除该节点，除此之外和持久化节点没有区别。官方注释说是可以用在leader或者锁的场景中。
> create -c
## TTL节点
带有存活时间的节点，简单来说就是当该节点下面没有子节点的话，超过了TTL指定时间后就会被自动删除，特性跟上面的容器节点很像，只是容器节点没有超时时间而已。默认禁用，只能通过系统配置`zookeeper.extendedTypesEnabled=true`开启，不稳定。
> create -t
# 节点Znode操作
## 新增节点
```bash
create [‐s] [‐e] [‐c] [‐t ttl] path [data] [acl]
```
* 新增持久化序号节点
```bash
create -s /node
```
> Created /node0000000001
* 新增临时节点
```bash
create -e /node1
```
> Created /node1
* 新增持久化节点，并添加数据
```bash
create /node2 node-data
```
> Created /node2

## 删除节点
* 删除指定节点（不能包含子节点）
```bash
delete path
```
* 递归删除节点及其下级所有节点
```bash
deleteall path
```

## 查询（子）节点
* 查询指定节点的子节点
```bash
ls path
```
> ls /zookeeper
[config, quota]
* 递归查询节点及其下级所有节点
```bash
ls -R path
```
> ls -R /
/
/zookeeper
/zookeeper/config
/zookeeper/quota

## 节点数据
* 设置（修改）节点数据
```bash
set path node-data
```
> set /zookeeper zookeeper
* 获取节点数据
```bash
get [-s] path
```
> get /zookeeper
zookeeper
-s代表同时获取节点状态

## 节点状态
```bash
stat path
```
> stat /zookeeper
cZxid = 0x0 —— 创建Znode的事务ID
ctime = Thu Jan 01 08:00:00 CST 1970 —— Znode的创建时间
mZxid = 0x34 —— 最后修改Znode的事务ID
mtime = Mon Jun 20 22:32:50 CST 2022 —— Znode的最后修改时间
pZxid = 0x0 —— 最后添加或删除子节点的事务ID（子节点列表发生变化才会发生改变）
cversion = -2 —— Znode的子节点结果集版本（子节点增加、删除都会影响这个版本）
dataVersion = 2 —— Znode的数据版本
aclVersion = 0 —— Znode的ACL版本
ephemeralOwner = 0x0 —— 临时节点所有者的 session ID（零代表为非临时节点）。
dataLength = 9 —— Znode数据字段的长度
numChildren = 2 —— Znode子节点的数量





















