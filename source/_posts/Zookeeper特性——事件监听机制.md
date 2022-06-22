---
title: Zookeeper特性——事件监听机制
tags: ["Zookeeper", "事件监听机制", "一次性监听"]
categories: ["Zookeeper"]
---
Zookeeper有两大特性——文件系统和事件监听机制，本篇介绍事件监听机制。
# 事件类型
Zookeeper中的事件类型有五种，如下：
```java
public enum EventType {
    None (-1),
    NodeCreated (1),            // 当监听节点被创建时触发
    NodeDeleted (2),            // 当监听节点被删除时触发
    NodeDataChanged (3),        // 当监听节点的节点数据发生变化时触发
    NodeChildrenChanged (4);    // 当监听节点的子节点列表发生变化时触发
}
```
# 监听类型
Zookeeper的监听类型总体分为两类，节点数据监听（NodeDataChanged）和节点目录监听（NodeCreated、NodeDeleted、NodeChildrenChanged）。
## 节点数据监听
节点数据监听只能针对单个节点设置。当节点的节点数据发生变化时，Zookeeper服务端会向所有注册了该节点数据监听的客户端发送一个NodeDataChanged事件。
> 注意：节点数据并不单指节点存储的数据，还包括节点的元数据（状态信息）。
## 节点目录监听
节点目录监听对应三种事件，所以我们也可以将节点目录监听分为三类：
* 节点创建监听（NodeCreated）
> 当一个节点被创建时，Zookeeper服务端会向所有注册了该节点创建监听的客户端发送一个NodeCreated事件。这里可能会有疑问，如何向一个不存在的节点注册监听？确实，Zookeeper原生客户端不支持这种操作，但是Zookeeper的Java客户端的一些API是支持的。
* 节点删除监听（NodeDeleted）
> 当一个节点被删除时，Zookeeper服务端会向所有注册了该节点删除监听的客户端发送一个NodeDeleted事件。
* 子节点列表监听（NodeChildrenChanged）
> 当一个节点发生子节点数量变化（新增、删除）时，Zookeeper服务端会向所有注册了该子节点列表监听的客户端发送一个NodeChildrenChanged事件。

> 注意，节点目录监听不仅可以针对单个节点设置，还可以递归对节点及其所有下级节点进行设置。

# 监听操作（原生API）
## 增加监听
* 增加节点数据监听
```bash
get -w path
stat -w path
```
![节点数据监听.png](节点数据监听.png)
* 增加节点目录监听
```bash
ls -w path
```
![节点目录监听1.png](节点目录监听1.png)
* 递归增加节点目录监听
```bash
ls -R -w path
```
![节点目录监听2.png](节点目录监听2.png)
这里执行了5条命令，每条命令都有其意义所在：
    * ls -R -w /：列出所有节点，并给所有节点增加节点目录监听。也就是说有且仅有这7个节点被注册了节点目录监听。
    * delete /test/a：触发了`/test/a`节点的NodeDeleted事件；触发了`/test`节点的NodeChildrenChanged事件。
    * delete /test/b：触发了`/test/b`节点的NodeDeleted事件；但是没有触发`/test`节点的NodeChildrenChanged事件，因为上一步已经触发过了。
    * create /test/c：新增一个节点，目的是为了测试新节点是否会被注册事件监听。
    * create /test/c/1：没有触发`/test/c`节点的NodeChildrenChanged事件，证明新节点不会被注册事件监听。

## 移除监听
```bash
removewatches path
```




















