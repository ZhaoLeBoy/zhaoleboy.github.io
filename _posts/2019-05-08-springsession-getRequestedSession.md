---
layout:     post
title:      "2.1.SpringSession源码:getRequestedSession()"
subtitle:   "2.1.SpringSession源码:getRequestedSession()"
date:        2019-05-07
author:     "ZhaoLe"
header-img: "img/spring-session/post-bg.png"
catalog: true
tags:
    - SpringSession
---


```java
private S getRequestedSession() {
	if (!this.requestedSessionCached) {
		List<String> sessionIds = SessionRepositoryFilter.this.httpSessionIdResolver
				.resolveSessionIds(this);
		for (String sessionId : sessionIds) {
			if (this.requestedSessionId == null) {
				this.requestedSessionId = sessionId;
			}
			S session = SessionRepositoryFilter.this.sessionRepository.findById(sessionId);
			if (session != null) {
				this.requestedSession = session;
				this.requestedSessionId = sessionId;
				break;
			}
		}
		this.requestedSessionCached = true;
	}
	return this.requestedSession;
}
```
* 【3】获取sessionId，SpringSession对于sessionId获取的两种策略，一种从Cookie中获取(`CookieHttpSessionIdResolver`)，也是默认方式.另一种从Header中获取`HeaderHttpSessionIdResolver`,顺便说下，如果要还一种解析方式，需要预先配置下.
	*  用户代码如下:
		```java
		@Configuration
		public class SessionIdResolverConfig {
				@Bean
				public HttpSessionIdResolver httpSessionIdResolver() {
						return HeaderHttpSessionIdResolver.xAuthToken();
				}
		}
		```
	接着说`resolveSessionIds`,从`CookieHttpSessionIdResolver#resolveSessionIds`方法进去，内部有个`readCookieValues`,	直接从request中获取cookies然后循环，找到名称是`SESSION`，默认进行Base64解码，最后将sessionId放入列表中。
		
		```java
			//DefaultCookieSerializer.java
			@Override
			public List<String> readCookieValues(HttpServletRequest request) {
				Cookie[] cookies = request.getCookies();
				List<String> matchingCookieValues = new ArrayList<>();
				if (cookies != null) {
					for (Cookie cookie : cookies) {
						if (this.cookieName.equals(cookie.getName())) {
							String sessionId = (this.useBase64Encoding
									? base64Decode(cookie.getValue())
									: cookie.getValue());
							if (sessionId == null) {
								continue;
							}
							//TODO:jvmRoute不知道是什么。后期研究下
							if (this.jvmRoute != null && sessionId.endsWith(this.jvmRoute)) {
								sessionId = sessionId.substring(0,
										sessionId.length() - this.jvmRoute.length());
							}
							matchingCookieValues.add(sessionId);
						}
					}
				}
				return matchingCookieValues;
			}
		```
	
* 【5-15】循环从`httpSessionIdResolver`解析出来的sessionId列表，并且根据sessionId从存储(这里是Redis)中获取Session。顺序是RedisOperationsSessionRepository#findById -> RedisOperationsSessionRepository#getSession
  ```java
  	private RedisSession getSession(String id, boolean allowExpired) {
		Map<Object, Object> entries = getSessionBoundHashOperations(id).entries();
		if (entries.isEmpty()) {
			return null;
		}
		MapSession loaded = loadSession(id, entries);
		if (!allowExpired && loaded.isExpired()) {
			return null;
		}
		RedisSession result = new RedisSession(loaded);
		result.originalLastAccessTime = loaded.getLastAccessedTime();
		return result;
	}
  ```
  * 【2】从redis根据sessionId读取信息
  * 【6】本质上就是将读取出来的Map结构转换为自定义的`MapSession`

* 【16】获取session成功，顺便就把`requestedSessionCached`标记下已缓存





