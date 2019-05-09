---
layout:     post
title:      "2.SpringSession源码:流程中几个重要方法"
subtitle:   "2.SpringSession源码:流程中几个重要方法"
date:        2019-05-07
author:     "ZhaoLe"
header-img: "img/spring-session/post-bg.png"
catalog: true
tags:
    - SpringSession
---

接着上篇文章从`SessionRepositoryFilter.java`开始说起。之前说`SessionRepositoryFilter`过滤器将request和response封装成自己定义的类型。所以用户代码`request.getSession()`其实调用到的是`SessionRepositoryRequestWrapper`中已经被重写了的`getSession()`方法。


```java
//SessionRepositoryFilter
@Override
public HttpSessionWrapper getSession(boolean create) {
    HttpSessionWrapper currentSession = getCurrentSession();
    if (currentSession != null) {
        return currentSession;
    }
    S requestedSession = getRequestedSession();
    if (requestedSession != null) {
        if (getAttribute(INVALID_SESSION_ID_ATTR) == null) {
            requestedSession.setLastAccessedTime(Instant.now());
            this.requestedSessionIdValid = true;
            currentSession = new HttpSessionWrapper(requestedSession, getServletContext());
            currentSession.setNew(false);
            setCurrentSession(currentSession);
            return currentSession;
        }
    }
    else {
        if (SESSION_LOGGER.isDebugEnabled()) {
            SESSION_LOGGER.debug(...);
        }
        setAttribute(INVALID_SESSION_ID_ATTR, "true");
    }
    if (!create) {
        return null;
    }
    if (SESSION_LOGGER.isDebugEnabled()) {
        SESSION_LOGGER.debug(...);
    }
    S session = SessionRepositoryFilter.this.sessionRepository.createSession();
    session.setLastAccessedTime(Instant.now());
    currentSession = new HttpSessionWrapper(session, getServletContext());
    setCurrentSession(currentSession);
    return currentSession;
}
```

* 【4】 `getCurrentSession()`先根据名称从request中获取session，同一个请求中，session如果存在就直接获取，不需要再从存储中获取了
 
    ```java
    private HttpSessionWrapper getCurrentSession() {
        return (HttpSessionWrapper) getAttribute(CURRENT_SESSION_ATTR);
    }
    ```
  

* 【8】 `getRequestedSession()`从缓存中获取session，具体参照[【2.1.SpringSession源码:getRequestedSession】](!http://jinlipool.com/2019/05/07/springsession-getRequestedSession/)
* 【9-18】获取session成功后(此时的RequestedSession类型)，包装成HttpSessionWrapper类型，如果继续个跟进代码会发现 这些都是在`HttpSessionAdapter.java`适配器中进行的。然后调用`setCurrentSession`放回当前的请求中
* 【31】`createSession()` ，如果request和redis中都没有session 就要新创建一个了。 具体参照[【2.2SpringSession源码:createSession()】](!http://jinlipool.com/2019/05/07/springsession-createSession/)
* 【32-34】在RedisSession基础上再次封装下，然后存入当前请求中

最后在`SessionRepositoryFilter.java`里面的`doFilterInternal`方法里有个`commitSession`方法

```java
//SessionRepositoryFilter.java
private void commitSession() {
    HttpSessionWrapper wrappedSession = getCurrentSession();
    if (wrappedSession == null) {
        if (isInvalidateClientSession()) {
            SessionRepositoryFilter.this.httpSessionIdResolver.expireSession(this,
                    this.response);
        }
    }
    else {
        S session = wrappedSession.getSession();
        clearRequestedSessionCache();
        SessionRepositoryFilter.this.sessionRepository.save(session);
        String sessionId = session.getId();
        if (!isRequestedSessionIdValid()
                || !sessionId.equals(getRequestedSessionId())) {
            SessionRepositoryFilter.this.httpSessionIdResolver.setSessionId(this,
                    this.response, sessionId);
        }
    }
}
```
* 【3】直接从当前请求中获取session
* 【4-9】如果不存在，检查是否失效，如果失效就调用`CookieHttpSessionIdResolver#expireSession`将对应的Cookie值设置为空并写回响应
* 【12】finally阶段，清除掉一些缓存标记，初始化一些成员变量
* 【13】在`RedisOperationsSessionRepository#save`中实现，用于持久化，其中`session.saveDelta();`的调用在[【2.2SpringSession源码:createSession()】](!http://jinlipool.com/2019/05/07/springsession-createSession/)已经说明了
* 【14-19】在`CookieHttpSessionIdResolver#setSessionId`将sessionId写回response给客户端.

