---
layout:     post
title:      "1.SpringSession源码:从一个demo开始"
subtitle:   "1.SpringSession源码:从一个demo开始"
date:        2019-05-07
author:     "ZhaoLe"
header-img: "img/spring-session/post-bg.png"
catalog: true
tags:
    - SpringSession
---

* #### 版本号说明
>版本号：

* spring-session-data-redis：2.1.5.RELEASE
* spring-boot-starter-data-redis：2.1.4.RELEASE
* Spring Boot：2.1.4.RELEASE

* #### 配置文件

>pom.xml

```xml
<!--redis模块-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!--sessions模块-->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<!--web模块-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

>application.properties

```properties
spring.redis.host=127.0.0.1
spring.session.store-type=redis
server.port=8899
```

* #### 示例代码
直接在Controller层写个demo。然后运行起来。直接访问`http://localhost:8080/session` 就可以根据断点一步步调试。
>DemoController.java

```java
@ResponseBody
@RequestMapping(value = "/session")
public Map<String, Object> getSession(HttpServletRequest request) {
    request.getSession().setAttribute("username1", "guest1");
    Map<String, Object> map = new HashMap();
    map.put("sessionId", request.getSession().getId());
    map.put("maxInactiveInterval", request.getSession().getMaxInactiveInterval() / 60);
    map.put("createTime", new Date(request.getSession().getCreationTime()));
    return map;
}

@ResponseBody
@RequestMapping(value = "/get")
public String get(HttpServletRequest request) {
    String userName = (String) request.getSession().getAttribute("username");
    return userName;
}
```

### 核心类介绍

* #### SessionRepositoryFilter
Spring-Session中的`SessionRepositoryFilter`是核心过滤器。他用于将Http请求拦截然后包装成自己的请求。再进行后续的操作。

```java
//SessionRepositoryFilter.java
@Order(SessionRepositoryFilter.DEFAULT_ORDER)
public class SessionRepositoryFilter<S extends Session> extends OncePerRequestFilter {
...
}
```
使用`@Order`定义执行顺序,继承`OncePerRequestFilter`保证一次请求只通过一次filter

```java
//SessionRepositoryFilter.java
@Override
protected void doFilterInternal(HttpServletRequest request,
        HttpServletResponse response, FilterChain filterChain)
        throws ServletException, IOException {
    request.setAttribute(SESSION_REPOSITORY_ATTR, this.sessionRepository);

    SessionRepositoryRequestWrapper wrappedRequest = new SessionRepositoryRequestWrapper(
            request, response);
    SessionRepositoryResponseWrapper wrappedResponse = new SessionRepositoryResponseWrapper(
            wrappedRequest, response);
}
```
filter内部方法`doFilterInternal`中使用自定义的SessionRepositoryRequestWrapper,SessionRepositoryResponseWrapper包装下request和response。并且重写一些方法，`HttpServletRequest`中重要方法之一的`getSession`就进行重写了，如果跟着断点来，`getSession`也是必经的方法，具体详情我们后面在说。

一个`doFilterInternal`方法中包含了后面几个重要的类，先放上几张关系图，算是混个眼熟。

* #### SessionRepositoryRequestWrapper和SessionRepositoryResponseWrapper

>封装的Request和Response有关的关系图

![1.png](/img/spring-session/1.png)

* #### HttpSessionWrapper.java

>HttpSessionWrapper的关系图，本质上它就是对HttpSession的一个扩展替代

![1.png](/img/spring-session/2.png)

* `Session.java`作为第顶级接口,提供了一些常规化的session操作，跟HttpSession方法大致相同
* `HttpSessionAdapter.java`适配器实现了`HttpSession`中的方法，将SpringSession的`Session.java`中方法适配servlet的`HttpSession.java`中的方法
* `HttpSessionWrapper.java`允许从session实例冲创建HttpSession。也是对session的一层包装。

* #### RedisOperationsSessionRepository.java
>SpringSession的持久化,提供持三种久化方式jdbc,redis,hazelcast,后面只看redis这一种持久化方式。

![3.png](/img/spring-session/3.png)
    
* `SessionRepository.java` 用于管理session实例的存储,也是一些常规的操作接口。
* `FindByIndexNameSessionRepository.java`是对SessionRepository的扩展， 获取session用。
* `MessageListener.java`是跟redis消息监听有关。
* `RedisOperationsSessionRepository.java`是具体实现上面接口方法的类。




