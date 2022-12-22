---
title: Spring Security——自定义登录页面
tags: ["Spring", "Spring Security", "认证授权"]
categories: ["Spring Security"]
---
# 相关页面准备
`src/main/resources/static/`下准备如下页面：
登录页
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>登录页</title>
</head>
<body>
    <form action="/user/login" method="post">
        用户名：<input type="text" name="username"><br/>
        密码：<input type="password" name="password"><br/>
        <input type="submit" value="提交"/>
    </form>
</body>
</html>
```
<!-- more -->
主页
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>首页</title>
</head>
<body>
登录成功！
</body>
</html>
```
登陆失败页
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>首页</title>
</head>
<body>
登录失败！<a href="/login.html">重新登录</a>
</body>
</html>
```
# HttpSecurity配置
```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private MyUserDetailsService myUserDetailsService;

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // bcrypt加密
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(myUserDetailsService);
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 认证
        http.formLogin()                                // 表单提交
                .usernameParameter("username")          // 指定用户名对应的input框的name属性值，默认为username
                .passwordParameter("password")          // 指定密码对应的input框的name属性值，password
                .loginPage("/login.html")               // 自定义登录页面
                .loginProcessingUrl("/user/login")      // 登录请求路径，自定义，但必须和表单的action属性值保持一致
                .defaultSuccessUrl("/main.html")        // a.登录成功后默认重定向的地址（记忆原始请求路径）
                .failureUrl("/error.html");             // a.登录失败后重定向的地址
//                .successForwardUrl("/login/toMain")     // b.登录成功后转发的地址
//                .failureForwardUrl("/login/toError");   // b.登录失败后转发的地址
//                .successHandler(new MyAuthenticationSuccessHandler("/main.html"))   // c.自定义登录成功处理器
//                .failureHandler(new MyAuthenticationFailureHandler("/error.html")); // c.自定义登录失败处理器

        // 授权
        http.authorizeRequests()
                .antMatchers("/login.html", "/user/login", "/error.html", "/login/toError").permitAll() // 不需要登录认证的请求
                .anyRequest().authenticated()   // 其他请求全部需要登录认证
                .and().csrf().disable();        // 禁用跨站请求伪造保护
    }
}
```
# 登录测试
![tryLogin5.png](tryLogin5.png)
登录成功：
![hello2.png](hello2.png)
# defaultSuccessUrl方法解析
`defaultSuccessUrl(String url)`和`failureUrl(String url)`指定了登录成功或者失败后**重定向**的地址，我们以`defaultSuccessUrl(String url)`为例进行分析：
```java
public final T defaultSuccessUrl(String defaultSuccessUrl) {
    return defaultSuccessUrl(defaultSuccessUrl, false);
}
```
```java
public final T defaultSuccessUrl(String defaultSuccessUrl, boolean alwaysUse) {
    // 这一行是关键，指定了登录成功处理器——保存了请求意识的登录成功处理器
    SavedRequestAwareAuthenticationSuccessHandler handler = new SavedRequestAwareAuthenticationSuccessHandler();
    handler.setDefaultTargetUrl(defaultSuccessUrl);
    handler.setAlwaysUseDefaultTargetUrl(alwaysUse);
    this.defaultSuccessHandler = handler;
    return successHandler(handler);
}
```
```java
public class SavedRequestAwareAuthenticationSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {

	protected final Log logger = LogFactory.getLog(this.getClass());

	private RequestCache requestCache = new HttpSessionRequestCache();

	@Override
	public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws ServletException, IOException {
		SavedRequest savedRequest = this.requestCache.getRequest(request, response); // 原始请求
		if (savedRequest == null) {
			super.onAuthenticationSuccess(request, response, authentication);
			return;
		}
		String targetUrlParameter = getTargetUrlParameter();
		if (isAlwaysUseDefaultTargetUrl()
				|| (targetUrlParameter != null && StringUtils.hasText(request.getParameter(targetUrlParameter)))) {
			this.requestCache.removeRequest(request, response);
			super.onAuthenticationSuccess(request, response, authentication);
			return;
		}
		clearAuthenticationAttributes(request);
		String targetUrl = savedRequest.getRedirectUrl(); // 原始请求路径
		getRedirectStrategy().sendRedirect(request, response, targetUrl);
	}

	public void setRequestCache(RequestCache requestCache) {
		this.requestCache = requestCache;
	}

}
```
> 这里需要注意的一点就是`defaultSuccessUrl(String url)`会记忆原始请求，并在登录成功后自动重定向到我们初始访问时的请求路径。
# successForwardUrl方法解析
`successForwardUrl(String url)`和`failureForwardUrl(String url)`指定了登录成功或者失败后**转发**的地址，我们以`successForwardUrl(String url)`为例进行分析：
```java
public FormLoginConfigurer<H> successForwardUrl(String forwardUrl) {
    // 这一行是关键，指定了登录成功处理器——转发处理器
    successHandler(new ForwardAuthenticationSuccessHandler(forwardUrl));
    return this;
}
```
```java
public class ForwardAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

	private final String forwardUrl;

	/**
	 * @param forwardUrl
	 */
	public ForwardAuthenticationSuccessHandler(String forwardUrl) {
		Assert.isTrue(UrlUtils.isValidRedirectUrl(forwardUrl), () -> "'" + forwardUrl + "' is not a valid forward URL");
		this.forwardUrl = forwardUrl;
	}

	@Override
	public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication) throws IOException, ServletException {
		request.getRequestDispatcher(this.forwardUrl).forward(request, response); // 直接转发请求到指定路径
	}

}
```
# 自定义登录成功/失败处理器
## 登录成功处理器
```java
public class MyAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

    private String redirectUrl;

    public MyAuthenticationSuccessHandler(String redirectUrl) {
        this.redirectUrl = redirectUrl;
    }

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        // 重定向
        response.sendRedirect(redirectUrl);
    }

}
```
## 登录失败处理器
```java
public class MyAuthenticationFailureHandler implements AuthenticationFailureHandler {

    private String redirectUrl;

    public MyAuthenticationFailureHandler(String redirectUrl) {
        this.redirectUrl = redirectUrl;
    }

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        // 重定向
        response.sendRedirect(redirectUrl);
    }
}
```