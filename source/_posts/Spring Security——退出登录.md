---
title: Spring Security——退出登录
tags: ["Session", "会话管理"]
categories: ["Spring Security"]
---
Spring Security默认实现了退出登录的逻辑，只需要调用`/logout`接口即可。默认的退出url为/logout，退出成功后跳转到/login?logout。
```java
public final class LogoutConfigurer<H extends HttpSecurityBuilder<H>>
		extends AbstractHttpConfigurer<LogoutConfigurer<H>, H> {

	private String logoutSuccessUrl = "/login?logout";

	private String logoutUrl = "/logout";
}
```
<!-- more -->
我们可以自定义退出逻辑，配置如下：
```java
http.logout()
    .logoutUrl("/logout")
    .logoutSuccessUrl("/login?logout");
```
当退出登录时，将发生：
1. 销毁Session
2. 清除认证状态
3. 重定向到登录页

```java
public class LogoutFilter extends GenericFilterBean {
    private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		if (requiresLogout(request, response)) {
			Authentication auth = SecurityContextHolder.getContext().getAuthentication();
			if (this.logger.isDebugEnabled()) {
				this.logger.debug(LogMessage.format("Logging out [%s]", auth));
			}
			this.handler.logout(request, response, auth); // 退出登录
			this.logoutSuccessHandler.onLogoutSuccess(request, response, auth); // 退出登录成功处理
			return;
		}
		chain.doFilter(request, response);
	}
}
```
```java
public class SecurityContextLogoutHandler implements LogoutHandler {
    @Override
	public void logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication) {
		Assert.notNull(request, "HttpServletRequest required");
		if (this.invalidateHttpSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				session.invalidate(); // 销毁Session
				if (this.logger.isDebugEnabled()) {
					this.logger.debug(LogMessage.format("Invalidated session %s", session.getId()));
				}
			}
		}
		SecurityContext context = SecurityContextHolder.getContext();
		SecurityContextHolder.clearContext();
		if (this.clearAuthentication) {
			context.setAuthentication(null); // 清除认证状态
		}
	}
}
```
```java
public abstract class AbstractAuthenticationTargetUrlRequestHandler {
    protected void handle(HttpServletRequest request, HttpServletResponse response, Authentication authentication)
			throws IOException, ServletException {
		String targetUrl = determineTargetUrl(request, response, authentication);
		if (response.isCommitted()) {
			this.logger.debug(LogMessage.format("Did not redirect to %s since response already committed.", targetUrl));
			return;
		}
		this.redirectStrategy.sendRedirect(request, response, targetUrl); // 重定向到登录页
	}
}
```
