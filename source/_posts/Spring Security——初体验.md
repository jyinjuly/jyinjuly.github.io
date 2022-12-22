---
title: Spring Security——初体验
tags: ["Spring", "Spring Security", "认证授权"]
categories: ["Spring Security"]
---

关于Spring Security的介绍我们不做过多赘述，下面是Spring官网对其描述：
> Spring Security is a powerful and highly customizable authentication and access-control framework.  It is the de-facto standard for securing Spring-based applications.
> Spring Security是一个功能强大且高度可定制的身份验证和访问控制框架。它是保护基于spring的应用程序的事实上的标准。

我们直接开始Spring Security的使用（注：本篇只介绍入门使用，旨在让大家对Spring Security有一个简单的认知）！
> 本系列均以与SpringBoot的集成为例进行演示
<!-- more -->

# 接口准备
创建SpringBoot工程，仅引入Web模块：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
然后提供一个接口：
```java
@RestController
@RequestMapping("/test")
public class TestController {

    @GetMapping("/hello")
    public String hello() {
        return "hello world.";
    }

}
```

# 未集成Spring Security的接口访问
浏览器访问接口`http://localhost:8080/test/hello`，直接返回响应结果：
![hello1.png](hello1.png)
# 集成Spring Security的接口访问
然后我们引入Security模块：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
再次浏览器访问接口`http://localhost:8080/test/hello`，结果如下：
![login.png](login.png)
可以看到，我们的接口已经被保护起来了！想要访问，就必须进行登录。而登录需要用户名和密码，下面我们介绍几种获取用户名和密码的方式。
## 默认用户名密码
Spring Security提供了默认的用户名和密码，用户名固定为`user`，密码则是在SpringBoot启动时随机生成，并且打印到控制台了：
![DefaultPassword.png](DefaultPassword.png)
我们尝试登录：
![tryLogin1.png](tryLogin1.png)
登录成功：
![hello1.png](hello1.png)
这里我们注意一下默认密码是由`UserDetailsServiceAutoConfiguration`提供的，该类核心逻辑为：
```java
@AutoConfiguration
@ConditionalOnClass(AuthenticationManager.class)
@ConditionalOnBean(ObjectPostProcessor.class)
@ConditionalOnMissingBean(
		value = { AuthenticationManager.class, AuthenticationProvider.class, UserDetailsService.class, AuthenticationManagerResolver.class })
public class UserDetailsServiceAutoConfiguration {

	@Bean
	@Lazy
	public InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties,
			ObjectProvider<PasswordEncoder> passwordEncoder) {
		SecurityProperties.User user = properties.getUser();
		List<String> roles = user.getRoles();
		return new InMemoryUserDetailsManager(
				User.withUsername(user.getName()).password(getOrDeducePassword(user, passwordEncoder.getIfAvailable()))
						.roles(StringUtils.toStringArray(roles)).build());
	}
}
```
SpringBoot启动的时候，该类会自动装配一个`InMemoryUserDetailsManager`类型的Bean到Spring容器中，而该Bean则在其内存中维护了一个用户信息。该用户信息又来自配置类`SecurityProperties`，如下：
```java
@ConfigurationProperties(prefix = "spring.security")
public class SecurityProperties {

	private final User user = new User();

	public User getUser() {
		return this.user;
	}

	public static class User {

		/**
		 * Default user name.
		 */
		private String name = "user";

		/**
		 * Password for the default user name.
		 */
		private String password = UUID.randomUUID().toString();

	}

}
```
可以看到，默认用户名为user，默认密码是一个随机生成的UUID。
## 基于配置文件方式
知道了默认用户名和密码的本质，我们就理所应当地知道了可以通过配置文件的方式自定义用户名和密码：
```yml
spring:
  security:
    user:
      name: zhangsan
      password: 123456
```
我们尝试登录：
![tryLogin2.png](tryLogin2.png)
登录成功：
![hello1.png](hello1.png)
## 自定义UserDetailsService
我们再回过头去看下`UserDetailsServiceAutoConfiguration`，它有一个注解`@ConditionalOnMissingBean(value = { AuthenticationManager.class, AuthenticationProvider.class, UserDetailsService.class, AuthenticationManagerResolver.class })`。该注解的意思是当Spring容器中不存在`AuthenticationManager`、`AuthenticationProvider`、`UserDetailsService`、`AuthenticationManagerResolver`类型的Bean时，该自动装配类才会起作用。换句话说，如果Spring容器中存在四种中的任意一种，该类则不会起作用。我们看一下`UserDetailsService`：
```java
public interface UserDetailsService {

	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;

}
```
该类的作用是通过用户名加载用户信息。再进一步说明一下，有了该类就不再提供默认的用户名和密码了。换而言之，通过该类我们可以自定义用户信息：
```java
@Component
public class MyUserDetailsService implements UserDetailsService {

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // "{noop}"：不加密
        return User.withUsername("lisi").password("{noop}123456").authorities(AuthorityUtils.NO_AUTHORITIES).build();
    }

}
```
我们尝试登录：
![tryLogin3.png](tryLogin3.png)
登录成功：
![hello1.png](hello1.png)
## 自定义UserDetailsManager
其实`UserDetailsManager`本质上也是`UserDetailsService`，这点从Java类图就能看出来：
![UserDetailsManager.png](UserDetailsManager.png)
有时候我们也可以通过自定义`UserDetailsManager`来实现自定义用户名密码：
```java
@Component
public class MyUserDetailsManager implements UserDetailsManager, InitializingBean {

    private Map<String, UserDetails> users = new HashMap<>();

    @Override
    public void afterPropertiesSet() throws Exception {
        UserDetails user = User
                .withUsername("wangwu")
                .password("{noop}123456")
                .authorities(AuthorityUtils.NO_AUTHORITIES).build();
        users.put("wangwu", user);
    }

    @Override
    public void createUser(UserDetails user) {
        users.putIfAbsent(user.getUsername(), user);
    }

    @Override
    public void updateUser(UserDetails user) {
        users.put(user.getUsername(), user);
    }

    @Override
    public void deleteUser(String username) {
        users.remove(username);
    }

    @Override
    public void changePassword(String oldPassword, String newPassword) {

        Authentication currentUser = SecurityContextHolder.getContext().getAuthentication();
        if (currentUser == null) {
            // This would indicate bad coding somewhere
            throw new AccessDeniedException(
                    "Can't change password as no Authentication object found in context " + "for current user.");
        }
        String username = currentUser.getName();

        UserDetails userDetails = this.users.get(username);
        Assert.state(userDetails != null, "Current user doesn't exist in database.");
        // 自行实现修改密码逻辑
    }

    @Override
    public boolean userExists(String username) {
        return users.containsKey(username);
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return users.get(username);
    }

}
```
我们尝试登录：
![tryLogin4.png](tryLogin4.png)
登录成功：
![hello1.png](hello1.png)
## 基于配置类WebSecurityConfigurerAdapter
```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private MyUserDetailsService myUserDetailsService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 指定UserDetailsService
        auth.userDetailsService(myUserDetailsService);
        // 使用内存方式预置一个用户
//        auth.inMemoryAuthentication()
//                .withUser("lisi")
//                .password("{noop}123456")
//                .authorities(AuthorityUtils.NO_AUTHORITIES);
    }

}
```
我们尝试登录：
![tryLogin3.png](tryLogin3.png)
登录成功：
![hello1.png](hello1.png)
# Spring Security密码加密器
通常使用Spring Security时是需要对密码进行加密处理的，密码加密器的种类有很多：
```java
public final class PasswordEncoderFactories {

	private PasswordEncoderFactories() {
	}

	@SuppressWarnings("deprecation")
	public static PasswordEncoder createDelegatingPasswordEncoder() {
		String encodingId = "bcrypt";
		Map<String, PasswordEncoder> encoders = new HashMap<>();
		encoders.put(encodingId, new BCryptPasswordEncoder());
		encoders.put("ldap", new org.springframework.security.crypto.password.LdapShaPasswordEncoder());
		encoders.put("MD4", new org.springframework.security.crypto.password.Md4PasswordEncoder());
		encoders.put("MD5", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("MD5"));
		encoders.put("noop", org.springframework.security.crypto.password.NoOpPasswordEncoder.getInstance());
		encoders.put("pbkdf2", new Pbkdf2PasswordEncoder());
		encoders.put("scrypt", new SCryptPasswordEncoder());
		encoders.put("SHA-1", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("SHA-1"));
		encoders.put("SHA-256",
				new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("SHA-256"));
		encoders.put("sha256", new org.springframework.security.crypto.password.StandardPasswordEncoder());
		encoders.put("argon2", new Argon2PasswordEncoder());
		return new DelegatingPasswordEncoder(encodingId, encoders);
	}

}
```
其中`noop`代表不加密，推荐使用`bcrypt`，指定密码加密器的方式有两种。
## 通过前缀指定
```java
@Component
public class MyUserDetailsService implements UserDetailsService {

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        BCryptPasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder();
        String encodePwd = bCryptPasswordEncoder.encode("123456");
        return User.withUsername("lisi").password("{bcrypt}" + encodePwd).authorities(AuthorityUtils.NO_AUTHORITIES).build();
    }

}
```
## 通过注入Bean指定
往Spring容器中注册`BCryptPasswordEncoder`密码加密器：
```java
@SpringBootApplication
public class SpringSecurityStudyApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringSecurityStudyApplication.class, args);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // bcrypt加密
    }
}
```
Spring容器中但凡有`PasswordEncoder`类型的Bean，就会被作为默认的密码加密器：
```java
@Component
public class MyUserDetailsService implements UserDetailsService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return User.withUsername("lisi").password(passwordEncoder.encode("123456")).authorities(AuthorityUtils.NO_AUTHORITIES).build();
    }

}
```
