---
title: Spring Security——会话（Session）管理
tags: ["Session", "Spring Security", "会话管理"]
categories: ["Spring Security"]
---

用户认证通过后，为了避免之后的每次操作都进行重复认证，可将用户的信息保存在会话中。spring security提供会话管理，认证通过后可将身份信息放入SecurityContextHolder上下文，SecurityContext与当前线程进行绑定，方便获取用户身份。

# 获取用户信息
```java
@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping("/hello")
    public String hello() {
        return "hello world." + getUsername();
    }

    private String getUsername() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

        if (!authentication.isAuthenticated()) {
            return null;
        }

        Object principal = authentication.getPrincipal();
        String username;
        if (principal instanceof UserDetails) {
            username = ((UserDetails) principal).getUsername();
        } else {
            username = principal.toString();
        }
        return username;
    }
}
```
<!-- more -->
# 会话控制
我们可以通过以下选项准确控制会话何时创建以及Spring Security如何与之交互：
| 机制 | 描述 |
| ---- | ---- |
| always | 总是创建一个HttpSession |
| ifRequired | Spring Security 仅仅将会在必要的时候创建一个HttpSession（默认） |
| never | Spring Security将永远不会创建一个HttpSession。但是如果已经存在一个HttpSession的话，Spring Security将会使用它 |
| stateless | Spring Security将永远不会创建一个HttpSession，也不使用它。并且它会暗示不使用cookie，所以每个请求都需要重新进行身份认证。这种无状态架构适用于REST API及其无状态认证机制 |
```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED);
}
```
# 会话超时
可以在servlet容器中设置Session的超时时间，如下设置Session有效期为600s：
```yml
server:
  servlet:
    session:
      timeout: 600s
```
session超时时间最低为60s，参考Tomcat源码`TomcatServletWebServerFactory#configureSession`：
```java
private long getSessionTimeoutInMinutes() {
    Duration sessionTimeout = getSession().getTimeout();
    if (isZeroOrLess(sessionTimeout)) {
        return 0;
    }
    return Math.max(sessionTimeout.toMinutes(), 1);
}
```
session超时之后，可以通过Spring Security设置跳转的路径：
```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.sessionManagement().invalidSessionUrl("/session/invalid");

    // 还需要将"/session/invalid"放行
    http.authorizeRequests()
            .antMatchers("/login.html", "/user/login", "/error.html", "/login/toError", "/session/invalid").permitAll()
            .anyRequest().authenticated()
            .and().csrf().disable();
}
```
```java
@RestController
@RequestMapping("/session")
public class SessionController {

    @GetMapping("/invalid")
    public String hello() {
        return "session invalid.";
    }

}
```
> 注意：重启完服务，首次访问`/login.html`时，由于此时服务器还没有session，因此会跳转到`/session/invalid`。同时首次请求，服务器将会创建一个空session（无用户信息），所以第二次访问时将会跳转到登录页。
# 会话并发控制
当用户在一个浏览器登录后，又尝试在另一个浏览器登录相同的账户。为了不让用户无限制的登录，可以有以下两种处理方法：
1. 后登录的将先登录的挤兑下线
2. 当登录到达上限后，禁止再登录

## 挤兑下线
```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin(); // 使用原始的登录页面

    http.sessionManagement()
            .maximumSessions(1);
}
```
测试方法：

1. 首先用Chrom浏览器登录账户liteng
2. 然后用FireFox浏览器登录账户liteng
3. 然后再用Chrom浏览器访问主页`/main.html`

测试结果：
`This session has been expired (possibly due to multiple concurrent logins being attempted as the same user).`这是`ResponseBodySessionInformationExpiredStrategy`的默认实现。
![SessionInformationExpiredStrategy.png](SessionInformationExpiredStrategy.png)
我们还可以自定义实现：
```java
public class MySessionInformationExpiredStrategy implements SessionInformationExpiredStrategy {

    @Override
    public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException, ServletException {
        HttpServletResponse response = event.getResponse();
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("您已被挤兑下线！");
    }

}
```
```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.sessionManagement()
            .maximumSessions(1)
            .expiredSessionStrategy(new MySessionInformationExpiredStrategy());
}
```
测试结果：
![挤兑下线.png](挤兑下线.png)
## 禁止登录
```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin(); // 使用原始的登录页面

    http.sessionManagement()
            .maximumSessions(1)
            .maxSessionsPreventsLogin(true);
}
```
测试方法：

1. 首先用Chrom浏览器访问主页`/main.html`，跳转登录页，登录账户liteng
2. 然后用FireFox浏览器访问主页`/main.html`，跳转登录页，登录账户liteng

测试结果：
![禁止登录.png](禁止登录.png)

# 分布式Session
实际生产中，一个服务往往是多实例的，并通过Nginx做负载均衡，将请求分发到不同的实例上。这就会存在分布式Session的问题：假设有1和2两个实例，由Nginx轮询分发请求。此时用户请求`/main.html`，触发登录，然后服务端将会创建Session。我们假设此次请求被分发到实例1上，理所应当地，Session也将会存储在实例1上。然后用户再次访问`/main.html`，这次的请求被分配到实例2上，由于实例2并不存在Session，因此会再次触发登录，这对于用户来说是不合理的现象。
![分布式Session1.png](分布式Session1.png)
解决这个问题的思路是用户登录的会话信息不再存储到某个具体的服务实例中，而是保存到另一个第三方介质中，比如redis、mongodb等。所有的服务实例都从这个第三方介质来获取Session信息。比如：用户在实例1上登录，然后将会话信息保存到redis中，用户的下次请求被分配到实例2上，实例2再从redis中检查Session是否已经存在，如果存在就不用再登录，可以直接访问服务了。
![分布式Session2.png](分布式Session2.png)
具体做法如下：
1. 引入Spring Session依赖
```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
2. 配置Session存储方式
```yml
spring:
  session:
    store-type: redis
  redis:
    host: 192.168.175.175
    port: 6379
```

测试分布式Session：
1. 8080端口启动服务
2. 8081端口启动服务
3. 访问`http://localhost:8080/main.html`，触发登录
4. 访问`http://localhost:8081/main.html`，直接访问