---
layout: post
title: "一种在SpirngMVC3中防御CSRF攻击的实现"
date: 2013-09-29 09:36
comments: true
categories: Spring CSRF Security
---
关于CSRF是什么东西，请参见我的博文：[《浅析CSRF》](http://javanlu.github.io/blog/2013/09/27/csrf-brief-analysis/)。本文将给出一种在 SpringMVC3+Velocity 的框架下防御CSRF攻击的解决方案。主要思想参考 [EYAL LUPU](http://blog.eyallupu.com/2012/04/csrf-defense-in-spring-mvc-31.html)<sup>[1]</sup> 的解决方案，但使用Velocity宏来进行Token值的传递，而不是使用``spring form``标签。

<!--more-->

## <strong>一、方案概要</strong>

在服务器端生成私有的会话级token，使用Velocity的宏命令在客户端的Form表单中插入这个token；在表单提交后对，服务器端对token值进行校验，根据结果来区分该次请求是否合法：正确就放行，不正确就就阻断请求。

在这里，我们约定：**凡是更新资源的操作，都通过POST向服务器端发送**。这也是符合HTTP规范的约定。

## <strong>二、Token生成</strong>

使用一个 CSRFTokenManager 类来进行token生成的工作，代码如下：

{% codeblock CSRFTokenManager.java lang:java https://github.com/JavanLu/javan-blog/blob/master/springmvc-csrf-velocity/src/main/java/com/javan/security/CSRFTokenManager.java code view %}

package com.javan.security;

import java.util.UUID;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

/**
 * A manager for the CSRF token for a given session. The {@link #getTokenForSession(HttpSession)} should used to obtain
 * the token value for the current session (and this should be the only way to obtain the token value).
 */
public final class CSRFTokenManager {

	/**
	 * The token parameter name
	 */
	static final String CSRF_PARAM_NAME = "xToken";

	/**
	 * The location on the session which stores the token
	 */
	public final static String CSRF_TOKEN_FOR_SESSION_ATTR_NAME = CSRFTokenManager.class.getName() + ".tokenval";

	public static String getTokenForSession(HttpSession session) {
		String token = null;
		// cannot allow more than one token on a session - in the case of two requests trying to
		// init the token concurrently
		synchronized (session) {
			token = (String) session.getAttribute(CSRF_TOKEN_FOR_SESSION_ATTR_NAME);
			if (null == token) {
				token = UUID.randomUUID().toString();
				session.setAttribute(CSRF_TOKEN_FOR_SESSION_ATTR_NAME, token);
			}
		}
		return token;
	}

	/**
	 * Extracts the token value from the session
	 * 
	 * @param request
	 * @return
	 */
	public static String getTokenFromRequest(HttpServletRequest request) {
		String token = request.getParameter(CSRF_PARAM_NAME);
		if (token == null || "".equals(token)) {
			token = request.getHeader(CSRF_PARAM_NAME);
		}
		return token;
	}

	private CSRFTokenManager() {
	};
}

{% endcodeblock %}

``getTokenForSession``方法用于检查``HTTP session``中是否存在CSRF token：存在则返回；不存在则生成并存储到session中，然后返回。

## <strong>三、Token依附到Form</strong>

由于我的前端使用 Velocity 进行渲染，并且不使用 ``Spring form``标签，所以为了将token依附到Form表单中，我使用 Velocity 的宏指令进行渲染工作。

SpringMVC 中的配置，主要是配置 ``Velocity Tools``：

{% codeblock mvc-context.xml lang:xml https://github.com/JavanLu/javan-blog/blob/master/springmvc-csrf-velocity/src/main/webapp/WEB-INF/mvc-context.xml code view %}
<bean id="velocityConfig"
	class="org.springframework.web.servlet.view.velocity.VelocityConfigurer">
		<property name="resourceLoaderPath" value="/WEB-INF/views/" />
		<property name="velocityProperties">
			<props>
				<prop key="input.encoding">utf-8</prop>
				<prop key="output.encoding">utf-8</prop>
			</props>
		</property>
		<!-- Velocity的配置文件 -->
		<property name="configLocation" value="/WEB-INF/velocity.properties" />
</bean>
<bean id="viewResolver"
	class="org.springframework.web.servlet.view.velocity.VelocityViewResolver">
		<property name="cache" value="true" />
		<property name="contentType" value="text/html;charset=utf-8" />
		<property name="exposeSpringMacroHelpers" value="true" />
		<property name="suffix">
			<value>.vm</value>
		</property>
		<!-- Velocity tools的配置文件 -->
		<property name="toolboxConfigLocation" value="WEB-INF/toolbox.xml" />
		<property name="exposeSessionAttributes" value="true" />
</bean>
{% endcodeblock %}

``Toolbox.xml``中定义生成 CSRF token 的 tool:

{% codeblock toolbox.xml lang:xml https://github.com/JavanLu/javan-blog/blob/master/springmvc-csrf-velocity/src/main/webapp/WEB-INF/toolbox.xml code view %}
<toolbox>
	<tool>
	    <key>csrfTool</key>
	    <scope>application</scope>
	    <class>com.javan.util.CSRFTool</class>
	</tool>
</toolbox> 
{% endcodeblock %}


工具类CSRFTool.java：

{% codeblock CSRFTool.xml lang:xml https://github.com/JavanLu/javan-blog/blob/master/springmvc-csrf-velocity/src/main/java/com/javan/util/CSRFTool.java code view %}
package com.javan.util;

import javax.servlet.http.HttpServletRequest;

import com.javan.security.CSRFTokenManager;

public class CSRFTool {
	public static String getToken(HttpServletRequest request) {
		return CSRFTokenManager.getTokenForSession(request.getSession());
	}
}

{% endcodeblock %}

页面渲染CSRF token的宏命令：

{% codeblock macros-default.vm lang:js https://github.com/JavanLu/javan-blog/blob/master/springmvc-csrf-velocity/src/main/resources/macros-default.vm code view %}
#** CSRFToken
 * Generate a input field of type 'hidden' of token to avoid CSRF attack 
*#
#macro( CSRFToken $id)
	#if(!$id || $id == "")
		#set($id="xToken")
	#end
	<input type="hidden" id="$id" name="xToken" value="$csrfTool.getToken($request)"/>
#end
{% endcodeblock %}

在前端的Form表单中添加token值：

```html
<form id="userForm" action="/xxx.do" method="post">
	#CSRFToken()
	...
</form>
```

## <strong>四、验证Token</strong>

当请求提交后，在服务器端验证token值，在这里定义一个拦截器``CSRFHandlerInterceptor``:

{% codeblock CSRFHandlerInterceptor.java lang:java https://github.com/JavanLu/javan-blog/blob/master/springmvc-csrf-velocity/src/main/java/com/javan/security/CSRFHandlerInterceptor.java code view %}
package com.javan.security;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;
import org.springframework.web.servlet.resource.DefaultServletHttpRequestHandler;

/**
 * A Spring MVC <code>HandlerInterceptor</code> which is responsible to enforce CSRF token validity on incoming posts
 * requests. The interceptor should be registered with Spring MVC servlet using the following syntax:
 * 
 * <mvc:interceptors>
 *     <bean class="com.javan.security.CSRFHandlerInterceptor"/>
 * </mvc:interceptors>
 * 
 * @see CSRFRequestDataValueProcessor
 */
public class CSRFHandlerInterceptor extends HandlerInterceptorAdapter {

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

		if (handler instanceof DefaultServletHttpRequestHandler) {
			return true;
		}

		if (request.getMethod().equalsIgnoreCase("GET")) {
			// GET - allow the request
			return true;
		} else {
			// This is a POST request - need to check the CSRF token
			String sessionToken = CSRFTokenManager.getTokenForSession(request.getSession());
			String requestToken = CSRFTokenManager.getTokenFromRequest(request);
			if (sessionToken.equals(requestToken)) {
				return true;
			} else {
				response.sendError(HttpServletResponse.SC_FORBIDDEN, "Bad or missing CSRF value");
				return false;
			}
		}
	}
}
{% endcodeblock %}


在bean配置文件中配置：

```xml
<!-- CSRF Interceptor handlers -->
<mvc:interceptors>
	<bean class="com.javan.security.CSRFHandlerInterceptor" />
</mvc:interceptors>
```

## <strong>五、注意事项</strong>

如果项目``web.xml``中指定了``error-page``错误页面（比如403状态码对应的页面等），那么需要注意``CSRFHandlerInterceptor``拦截的问题。

如果验证成功，请求将沿着处理链继续传递；如果验证失败，请求将被暂停，发出一个``HTTP 403``的状态代码作为响应。但是问题来了，如果设置了``error-page``，Servlet 容器会先根据响应状态码将原始请求转发(forward)到具体的错误页面，然后发送到客户端浏览器。由于使用了拦截器，这次forward请求会再次被拦截，验证方法也会被触发，会再次验证失败。前端看不到403的错误提示页面。具体解决方案参见[这篇文章](http://blog.csdn.net/alphafox/article/details/8947117)<sup>[2]</sup>。

**源码已放到Github上，有需要的童鞋请移步** *[这里](https://github.com/JavanLu/javan-blog/tree/master/springmvc-csrf-velocity)*。


>### 参考资料
>[1] EYAL LUPU 的解决方案： [http://blog.eyallupu.com/2012/04/csrf-defense-in-spring-mvc-31.html](http://blog.eyallupu.com/2012/04/csrf-defense-in-spring-mvc-31.html)
>
>[2] 再谈Spring MVC中对于CSRF攻击的防御：[http://blog.csdn.net/alphafox/article/details/8947117](http://blog.csdn.net/alphafox/article/details/8947117)








