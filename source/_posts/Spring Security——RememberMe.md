---
title: Spring Security——RememberMe
tags: ["Session", "RememberMe", "会话管理"]
categories: ["Spring Security"]
---
# RememberMe简介
通常用户登录成功后，服务端会创建与之应的会话信息，也就是Session，用来保存其登录状态。但这个Session是有时效性的，比如5分钟。如果用户在这5分钟内没有任何操作的话，那么Session将会失效，用户再次访问则需要重新登录。诚然，这是出于安全考虑的一种设计。但有时候，这种频繁的登录会给我们带来苦恼。比如：PC通常只有个人使用。某天我们去浏览一个网站，然后起身上了个厕所，回来发现Sesssion失效了，重新登录一遍。过了一会，又起身接了杯水喝，回来发现Session又失效了，又得重新登录一遍，这是不是很麻烦？那么有没有一种办法，即使Session过期了，还能够保持登录状态呢？有，RememberMe就是为此而生的。RememberMe是一种服务器端的行为，其本质上和Session的类似，都是基于Cookie的实现的。

## 不使用RememberMe现象
不使用RememberMe，每次Session过期都需要重新登录，其现象如下：
![session1.png](session1.png)
然后我们登录，不勾选“记住我”，结果如下：
![session2.png](session2.png)
等待60s（我们配置的Session过期时间），再次访问，结果如下：
![session3.png](session3.png)

## 使用RememberMe现象
使用RememberMe，即使Session过期也不需要重新登录，其现象如下：
![session1.png](session1.png)
然后我们登录，勾选“记住我”，结果如下：
![session4.png](session4.png)
然后立刻再次访问（60s之内），结果如下：
![session5.png](session5.png)
60s之后（Session已过期），再次访问，结果如下：
![session6.png](session6.png)

# RememberMe原理
## remember-me生成规则
我们发现，当使用了RememberMe功能时，Cookie中多了一个`remember-me`。它同样是由服务端生成的，具体规则如下：
1. 登录成功，生成一条记录（包含用户名、随机生成的序列号、token）。
> 如果基于数据库存储RememberMe的话，如下：
![RememberMe.png](RememberMe.png)
2. 将username、series以及token加密处理，生成remember-me。

## RememberMe访问流程
1. 使用RememberMe登录，登录成功
2. 服务端生成Session和RememberMe
3. 浏览器Cookie存储Session和RememberMe
4. 浏览器访问资源，请求头携带Session和RememberMe
5. 服务端验证Session，如果验证成功（Session未过期），访问通过
6. 如果Session过期，则验证RememberMe，如果验证成功（RememberMe未过期），则访问通过
7. 服务端重新生成Session和RememberMe
8. 浏览器Cookie刷新Session和RememberMe
9. 如果RememberMe过期，则需要重新登录

