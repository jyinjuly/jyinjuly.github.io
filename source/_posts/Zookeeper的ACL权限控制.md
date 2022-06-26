---
title: Zookeeper的ACL权限控制
tags: ["Zookeeper", "ACL"]
categories: ["Zookeeper"]
---
Zookeeper的ACL权限控制，可以控制节点的读写操作，保证数据的安全性，Zookeeper的ACL权限设置分为3部分组成，分别是：权限模式（Scheme）、授权对象（ID）、权限信息（Permission）。最终组成一条例如“scheme:id:permission”格式的ACL请求信息。
# 权限模式
权限模式用来设置ZooKeeper服务器进行权限验证的方式。
## IP模式
ZooKeeper可以针对一个IP或者一段IP地址授予某种权限。比如我们可以让一个IP地址为“ip:192.168.0.110”的机器对服务器上的某个数据节点具有写入的权限。或者也可以通过“ip:192.168.0.1/24”给一段IP地址的机器赋权。
> ip:192.168.175.175:rw

<!-- more -->
## 口令模式
也可以理解为用户名密码的方式。在ZooKeeper中这种验证方式是Digest认证，而Digest这种认证方式首先在客户端传送“username:password”这种形式的权限表示符后，ZooKeeper服务端会对密码部分使用SHA-1和BASE64算法进行加密，以保证安全性。
## 超级管理员模式
Super可以认为是一种特殊的Digest认证。具有Super权限的客户端可以对ZooKeepers上的任意数据节点进行任意操作。
## World模式
world模式表示授权所有用户，一般搭配anyone使用。
> world:anyone:cwrda
# 授权对象
授权对象就是说我们要把权限赋予谁，而对应于不同的权限模式来说，授权对象有所不同。如下：
| IP模式 | 口令模式 | 超级管理员模式 | World模式 |
|--|--|--|--|
| ip地址/ip地址列表（多个ip地址以逗号隔开）/ip地址段 | 用户名 | 用户名 | 所有用户 |
# 权限信息
## 创建权限（c:create）
授予权限的对象可以在数据节点下**创建子节点**
## 删除权限（d:delete）
授予权限的对象可以删除该节点的**子节点**
## 读取权限（r:read）
授予权限的对象可以读取该节点的节点数据以及子节点的列表信息
## 更新权限（w:write）
授予权限的对象可以更新该节点的节点数据
## 管理权限（a:admin）
授予权限的对象可以对该节点体进行ACL权限设置
# 权限操作（API）
## getAcl
获取某个节点的acl权限信息
## setAcl
设置某个节点的acl权限信息
## addauth
输入认证授权信息，相当于登录
# ACL实操
## IP模式
```bash
create /test test-data ip:192.168.175.175:rw
```
> 创建节点，并授予ip地址为192.168.175.175的客户端该节点的读写权限
## 口令模式
口令模式首先需要生成口令，有两种方式：
* 通过命令直接生成
```bash
echo ‐n <user>:<password> | openssl dgst ‐binary ‐sha1 | openssl base64
```
* 通过Java API生成
```java
@Test
void testGenerateDigest() throws NoSuchAlgorithmException {
    log.info(DigestAuthenticationProvider.generateDigest("user1:password1")); // user1:XDkd2dsEuhc9ImU3q8pa8UOdtpI=
}
```

```bash
create /test2 test2-data digest:user1:XDkd2dsEuhc9ImU3q8pa8UOdtpI=:rw
```
> 创建节点，并授予用户user1该节点的读写权限

![口令模式.png](口令模式.png)

还有一种明文的口令模式，和digest用法一样，只需要在授权的时候把digest替换成auth即可。只不过需要在登录状态下才可以使用：
```bash
create /test3 test3-data auth:user1:password1
```

![明文口令模式.png](明文口令模式.png)
## 超级管理员模式
这是一种特殊的Digest模式， 在Super模式下超级管理员用户可以对Zookeeper上的任意节点进行任何操作。需要在启动时通过JVM系统参数开启：
```shell
‐Dzookeeper.DigestAuthenticationProvider.superDigest=user1:XDkd2dsEuhc9ImU3q8pa8UOdtpI=
```
> 其中user1为超级管理员用户名，password1为超级管理员密码

可以通过修改zkServer.sh来启用超级管理员：
![超级管理员模式.png](超级管理员模式.png)

## 跳过ACL认证
可以通过系统参数zookeeper.skipACL=yes进行配置，默认是no。可以配置为yes，则配置过的ACL将不再进行权限检测。


















