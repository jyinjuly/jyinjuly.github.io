---
title: Windows安装Nginx
tags: ["Nginx", "负载均衡", "反向代理"]
categories: ["Nginx"]
---

# 下载安装包
官网地址：`http://nginx.org/en/download.html`
![Nginx下载.png](Nginx下载.png)
<!-- more -->
# 解压Nginx
![解压Nginx.png](解压Nginx.png)
# 启动Nginx
## 快捷启动
直接双击Nginx目录下的nginx.exe，双击后一个黑色的弹窗一闪而过，启动就完成了
![Nginx快捷启动.png](Nginx快捷启动.png)
## 命令启动
cmd窗口下，切换到Nginx安装目录，输入start nginx，回车启动
![Nginx命令启动.png](Nginx命令启动.png)
## 启动验证
打开浏览器，输入`localhost`，回车，进入Nginx首页
![Nginx首页.png](Nginx首页.png)
# 关闭Nginx
cmd窗口下，切换到Nginx安装目录，输入以下命令，关闭Nginx
```bash
-- 快速停止nginx
nginx -s stop

-- 完整有序地停止nginx
nginx -s quit
```





















