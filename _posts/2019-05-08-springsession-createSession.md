---
layout:     post
title:      "2.2SpringSession源码:createSession()"
subtitle:   "2.2SpringSession源码:createSession()"
date:        2019-05-07
author:     "ZhaoLe"
header-img: "img/spring-session/post-bg.png"
catalog: true
tags:
    - SpringSession
---

创建session具体实现是在`RedisOperationsSessionRepository`中

```java
@Override
public RedisSession createSession() {
	RedisSession redisSession = new RedisSession();
	if (this.defaultMaxInactiveInterval != null) {
		redisSession.setMaxInactiveInterval(
				Duration.ofSeconds(this.defaultMaxInactiveInterval));
	}
	return redisSession;
}
```
* 【3】new出一个RedisSession对象，简简单单一行，里面的学问倒是挺多的。

```java
//RedisSession.java
RedisSession() {
	this(new MapSession());
	this.delta.put(CREATION_TIME_ATTR, getCreationTime().toEpochMilli());
	this.delta.put(MAX_INACTIVE_ATTR, (int) getMaxInactiveInterval().getSeconds());
	this.delta.put(LAST_ACCESSED_ATTR, getLastAccessedTime().toEpochMilli());
	this.isNew = true;
	this.flushImmediateIfNecessary();
}
```

```java
//RedisSession.java
private void flushImmediateIfNecessary() {
	if (RedisOperationsSessionRepository.this.redisFlushMode == RedisFlushMode.IMMEDIATE) {
		saveDelta();
	}
}
```

说`flushImmediateIfNecessary`方法之前，先要知道在SpringSession中将数据写入Redis实例有两种模式`ON_SAVE`(默认)和`IMMEDIATE`，可以再`application.properties`文件中配置，例如`spring.session.redis.flush-mode=immediate`，这两种的区别在于:

  * `ON_SAVE模式`:只有当主动调用的时候(SessionRepository#save)才会被触发,在web环境中通常是在提交HTTP响应的时候完成，所以它是默认模式
  * `IMMEDIATE模式`:尽可能快的写入redis，在创建新的session时候就调用(SessionRepository#createSession())或者在session上设置attribute的时候也是会立刻写入redis
  
所以在默认情况下`this.flushImmediateIfNecessary`中的`saveDelta`不会被执行到，但是最后在commit阶段会执行保存动作，所以我们就放在这边一起看了，`saveDelta`方法保存已更改的任何属性，并更新此session的到期日期。
      
```java
private void saveDelta() {
	String sessionId = getId();
	saveChangeSessionId(sessionId);
	if (this.delta.isEmpty()) {
		return;
	}
	getSessionBoundHashOperations(sessionId).putAll(this.delta);
	
	String principalSessionKey = getSessionAttrNameKey(FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME);
	String securityPrincipalSessionKey = getSessionAttrNameKey(SPRING_SECURITY_CONTEXT);
	
	if (this.delta.containsKey(principalSessionKey)	|| this.delta.containsKey(securityPrincipalSessionKey)) {
		if (this.originalPrincipalName != null) {
			String originalPrincipalRedisKey = getPrincipalKey(
					this.originalPrincipalName);
			RedisOperationsSessionRepository.this.sessionRedisOperations
					.boundSetOps(originalPrincipalRedisKey).remove(sessionId);
		}
		String principal = PRINCIPAL_NAME_RESOLVER.resolvePrincipal(this);
		this.originalPrincipalName = principal;
		if (principal != null) {
			String principalRedisKey = getPrincipalKey(principal);
			RedisOperationsSessionRepository.this.sessionRedisOperations
					.boundSetOps(principalRedisKey).add(sessionId);
		}
	}

	this.delta = new HashMap<>(this.delta.size());

	Long originalExpiration = (this.originalLastAccessTime != null)
			? this.originalLastAccessTime.plus(getMaxInactiveInterval()).toEpochMilli() : null;
	RedisOperationsSessionRepository.this.expirationPolicy
			.onExpirationUpdated(originalExpiration, this);
}
```
      
在`RedisSession`中维护着一个`cache`，是属于`MapSession`类型，里面存放着最新跟session有关的基本信息例如`creationTime`,`lastAccessedTime`和所有的`sessionAttrs`等。`RedisSession`本身也有一些成员变量如`originalLastAccessTime`，`originalSessionId`和`delta`等等，其中`delta`是个Map类型，用来存放当前一些变更的对象(delta符号在数学公式里就表示变化的量)，默认的取值是从`cache`来的

* 【2】获取`cache`中的id
* 【3】将cache中的id和原始的`originalSessionId`进行比较，如果不同并且不是新创建的session，就会将
`originalSessionId`和`originalExpiredKey`给替换掉。
* 【7】根据sessionId将`delta`的值持久化写入redis
* 【9-26】获取sessionAttr属性`PRINCIPAL_NAME_INDEX_NAME`和`SPRING_SECURITY_CONTEXT`并且对其处理,本质上也是做替换。
* 【28】已经处理+持久化完delta里面的参数了，可以进行初始化。
* 【30-31】刷新下过期时间，直接拿最后访问时间+最大过期间隔（默认30分钟）
* 【32-33】SpringSession有自己的过期时间处理策略，//TODO....
      

