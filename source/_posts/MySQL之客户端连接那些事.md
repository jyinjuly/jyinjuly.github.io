---
title: MySQL之客户端连接那些事
tags: ["MySQL", "客户端连接"]
categories: ["MySQL"]
---

# Root用户忘记密码
1. 配置文件——my.cnf中添加：skip-grant-tables，保存退出；
2. 重启mysql服务使配置文件生效：service mysqld restart；
3. mysql -uroot -p回车登录；
4. 使用mysql数据库：use mysql；
<!-- more -->
5. 将root密码置为空：update user set authentication_string = '' where user = 'root';
6. my.cnf中删掉步骤1的语句——skip-grant-tables，重启mysql服务；
7. 登录mysql，修改root用户密码：ALTER USER 'root'@'localhost' IDENTIFIED with mysql_native_password BY '${password}';

# MySQL 8 连接失败：caching sha2...
MySQL 8 之前的版本使用的密码加密规则是mysql_native_password，但是在mysql 8 则是caching_sha2_password，解决办法如下：
## 修改加密规则
```sql
ALTER USER 'root'@'%' IDENTIFIED BY 'password' PASSWORD EXPIRE NEVER;
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
```
## 刷新权限
```sql
FLUSH PRIVILEGES;
```

# MySQL 8 客户端无法使用root用户登录
## root登录mysql
```bash
$ mysql -uroot -p
```

## 添加普通用户
```sql
CREATE USER 'liteng'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
GRANT ALL PRIVILEGES ON *.* TO 'liteng'@'%';
```
