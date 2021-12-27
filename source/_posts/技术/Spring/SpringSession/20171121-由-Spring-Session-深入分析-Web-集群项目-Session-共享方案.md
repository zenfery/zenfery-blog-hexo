---
title: 由 Spring Session 深入分析 Web 集群项目 Session 共享方案
#permalink: session-share
categories:
  - JAVA
  - J2EE
date: 2017-05-18 06:22:22
updated: 2017-11-21 20:46:00
author: Zenfery
tags:
  - 分布式
  - Session
thumbnail:
excerpt: 在集群 Web 系统中，Session 会话基本上是属于肯定要使用到的技术，用户的每一次请求，可能会分配至不同的后端 Web 服务器实例，如何能保证用户的每一次请求都能正确读取当前会话保存的 Session 内容，即 Session 共享。本文将对常用的 Session 共享方案做一个简单的介绍，重点分析 Spring Session 方案的使用及深入理解分析。
---

在集群 Web 系统中，Session 会话基本上是属于肯定要使用到的技术，用户的每一次请求，可能会分配至不同的后端 Web 服务器实例，如何能保证用户的每一次请求都能正确读取当前会话保存的 Session 内容，即 Session 共享。本文将对常用的 Session 共享方案做一个简单的介绍，重点分析 Spring Session 方案的使用及深入理解分析。

## Session 共享方案介绍

### 基于 Cookie 实现
多数情况下， Session 存储在服务器端的内存中（关联一个 Cookie 键值来标识会话）；而使用纯 Cookie 来实现的 Session 共享方案，会将所有的会话内容存储在客户端浏览器的 Cookie 中，并且设置用来做 Session 共享 Cookie 的过期时间为浏览器关闭时间（即浏览器关闭时，自动删除这些 Cookie）。当然，保存到客户的的 Session 内容尽量使用一定的加密算法进行加密存储，以保证安全性。

#### 如何实现？
- 编写一个名字类似 SessionUtils 的一个工具类，实现类似 setSession(key, value) 和 getSession(key) 的工具方法；约定开发人员在使用到 Session 的地方使用此工具类。
- 基于过滤器对 Request 对象进行包装，将所有的请求均转到包装类来进行处理，好处是，开发人员无须关注细节，和正常开发无异。

#### 优点
- **原生支持，轻量级。** 并不需要增加其它组件就能实现。
- **易于理解学习。** 并不需要学习新的技术就能理解，只需理解 Cookie 即可。
- **减轻服务器端压力。** 将 Session 内容保存在客户端，将不会占用服务器端内存；服务器也不需要考虑 Session 的过期清除机制等各种处理机制。

#### 缺点
- **需开发。** 多数语言的 Web 框架并没有此机制的现成抽象实现；如 J2EE 的 Servlet 规范就没有实现；
- **带宽增加。** 在高并发请求环境中，由于所有的 Cookie 都会在每个请求中被携带至服务器端，增加了网络请求的流量，占用网络带宽。所以在高并发请求的的环境中，使用此方案，尽量在 Session 存储少量的数据，以减轻网络压力。
- **安全性低。** 即使将存储在客户端的 Session 进行加密，在客户端存储和网络传输中，信息都可能被窃取，从而增加敏感信息泄漏的风险。

### 基于 ip hash 或 session sticky 实现
基于 ip hash 和 session sticky 的方案原理均是为了将同一个客户端的请求都导向到后端的同一服务端实例。
ip hash 采用的是将同一个 ip 或 ip 段的请求导向到同一个服务实例。session sticky 采用的是记忆同一个客户端的会话请求的服务器实例，该客户端的请求，将会导向至同一个服务实例。从而达到共享假象的共享 Session 。session sticky 的原理图如下：
![session-sticky](../asset/img/session-share-sticky.png)

#### 优点
- **简单，无需开发。** 常用的 Web 服务器均实现该机制，仅需要按照官方文档进行配置即可。
- **服务压力小。** 多数情况下，均会使用反向代理服务器（nginx、apache等）来配置此功能，该类服务器性能比较高；压力无需服务实例来承担。

#### 缺点
- **负载不均衡。** ip hash 机制是将 ip 作 hash() 后作为路由的 key，若请求 ip 分布不均衡，有倾斜，则会导致部分访问大的 ip （多数网络环境都是局域网对应同一个出口ip）均导向至同一个后端服务实例。
- **瞬时故障。** 若后端某一实例出现异常故障，无法提供服务，此前所有正在请求该故障服务实例的用户，将会丢失 Session ，从而请求失败。

### 基于共享存储（内存）实现
将原来保存在各服务实例中的 Session 保存到实例共享的同一存储介质中（一般采用内存，速度快，如 redis、memcache）。这样的话，无论请求访问到哪个服务实例，Session 均会对共享存储进行读写，从而达到 Session 共享的目的。目前基于 Tomcat 实现了memcache 和 redis 的方案。
- **基于 memcache 实现。** 官方站点：[memcached-session-manage](https://github.com/magro/memcached-session-manager)。
- **基于 redis 实现。** 官方站点：[tomcat-redis-session-manager](https://github.com/jcoleman/tomcat-redis-session-manager)。
Spring 一直是解决集成方案的“大佬”，当然也会有其整合方案；它提供的 Spring Session 更加丰富的特性；不仅仅是支持 Http Session 共享方案。官方地址：[spring-session](http://projects.spring.io/spring-session/)。后续内容主要围绕 Spring Session 来展开讨论。

## Spring Session 如何使用？

Spring Session 实现了基于 redis 、GemFire、JDBC 的共享方案。此处主要分析基于 XML 和 Java Config 两种方式的使用方式，无论哪种方式，均基于几个核心的实现类：DelegatingFilterProxy、RedisHttpSessionConfiguration、RedisConnectionFactory。其中 DelegatingFilterProxy 和 RedisConnectionFactory 不是 Spring Session 的核心实现，Spring Session 使用到这两个类。
- **RedisHttpSessionConfiguration。** 创建核心 Filter : SessionRepositoryFilter，对 Http Session 进行封装，改造其使用 redis 等来进行存储 Session。此类还创建一些辅助处理 session 的实用类实例，如 Session 监听器。
- **DelegatingFilterProxy。** 代理核心 Filter : SessionRepositoryFilter。使其可被 Spring IOC 管理。
- **RedisConnectionFactory。** 连接 redis 连接池工厂。

#### 配置引入jar包
在 maven 的 pom.xml 加入项目依赖，版本选择相应的版本即可：
```xml
<dependencies>
        <!-- ... -->

        <dependency>
                <groupId>org.springframework.session</groupId>
                <artifactId>spring-session-data-redis</artifactId>
                <version>1.0.2.RELEASE</version>
                <type>pom</type>
        </dependency>
        <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-web</artifactId>
                <version>4.1.6.RELEASE</version>
        </dependency>
</dependencies>
```

#### 基于 XML 配置
*以下配置均为简化的关键配置，完整配置请自行补充。*
Spring 容器配置，此配置会自动生成默认名字为 springSessionRepositoryFilter 的核心 Filter：SessionRepositoryFilter。RedisConnecttionFactory可以使用任意实现。
```xml
<context:annotation-config/>
<bean class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration"/>

<bean class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory" />
```

web.xml 中主要配置：
```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
        /WEB-INF/spring/\*.xml
    </param-value>
</context-param>
<listener>
    <listener-class>
        org.springframework.web.context.ContextLoaderListener
    </listener-class>
</listener>

<filter>
    <filter-name>springSessionRepositoryFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
    <filter-name>springSessionRepositoryFilter</filter-name>
    <url-pattern>/\*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
    <dispatcher>ERROR</dispatcher>
</filter-mapping>
```

#### 基于 Java Config 配置方式
配置 SessionRepositoryFilter，并 Config 类能被 Spring 注解扫描：
```java
@EnableRedisHttpSession
public class Config {

        @Bean
        public JedisConnectionFactory connectionFactory() {
                return new JedisConnectionFactory();
        }
}
```
配置 DelegatingFilterProxy：
```java
public class Initializer extends AbstractHttpSessionApplicationInitializer {

        public Initializer() {
                super(Config.class);
        }
}
```

## Spring Session 源码解析

#### RedisHttpSessionConfiguration类
使用 @Configuration 注解的 RedisHttpSessionConfiguration 核心功能即为创建 SessionRepositoryFilter 实例，@Bean 注解的方法，生成的实例，默认名称为方法名字，此外为：springSessionRepositoryFilter。
```java
@EnableRedisHttpSession
@Bean
public <S extends ExpiringSession> SessionRepositoryFilter<? extends ExpiringSession>
  springSessionRepositoryFilter(SessionRepository<S> sessionRepository, ServletContext servletContext) {
  SessionRepositoryFilter<S> sessionRepositoryFilter = new SessionRepositoryFilter<S>(sessionRepository);
  sessionRepositoryFilter.setServletContext(servletContext);
  if(httpSessionStrategy != null) {
    sessionRepositoryFilter.setHttpSessionStrategy(httpSessionStrategy);
  }
  return sessionRepositoryFilter;
}
```

SessionRepositoryFilter 过滤器使用 SessionRepositoryRequestWrapper 和 SessionRepositoryResponseWrapper 来处理请求和响应；这两个类分别通过实现 HttpServletRequestWrapper 和 HttpServletResponseWrapper 来对 Requset 和 Response 做封装。SessionRepositoryFilter 指定了 httpSessionStrategy 策略，默认策略为使用 Cookie 来指定：
```java
public class SessionRepositoryFilter<S extends ExpiringSession> extends OncePerRequestFilter {
  // ...
  private MultiHttpSessionStrategy httpSessionStrategy = new CookieHttpSessionStrategy();
}
```

CookieHttpSessionStrategy 指定默认的 Session Cookie 的 cookieName为 SESSION：
```java
public final class CookieHttpSessionStrategy implements MultiHttpSessionStrategy, HttpSessionManager {
  // ...
  private String cookieName = "SESSION";
}
```

SessionRepositoryRequestWrapper 中获取到的 Session 类为 SessionRepositoryRequestWrapper.HttpSessionWrapper 内部类；
```java
private final class HttpSessionWrapper implements HttpSession {}
```

HttpSessionWrapper 持有类 ExpiringSession 类的实现，此处为 RedisOperationsSessionRepository.RedisSession。RedisSession 使用 redis 的 hash 结构来存储 Session 内容。缓存的 Session Key 值由 RedisOperationsSessionRepository.getKey() 方法指定，Session 中内容的属性的 key 由 RedisOperationsSessionRepository.getSessionAttrNameKey() 指定：
```java
public class RedisOperationsSessionRepository implements
    SessionRepository<RedisOperationsSessionRepository.RedisSession> {
  static final String BOUNDED_HASH_KEY_PREFIX = "spring:session:sessions:";
  static final String SESSION_ATTR_PREFIX = "sessionAttr:";

  static String getKey(String sessionId) {
    return BOUNDED_HASH_KEY_PREFIX + sessionId;
  }

  static String getSessionAttrNameKey(String attributeName) {
    return SESSION_ATTR_PREFIX + attributeName;
  }
}
```

#### DelegatingFilterProxy类
```java
@Override
protected void initFilterBean() throws ServletException {
  synchronized (this.delegateMonitor) {
    if (this.delegate == null) {
      // If no target bean name specified, use filter name.
      if (this.targetBeanName == null) {
        this.targetBeanName = getFilterName();
      }
      // Fetch Spring root application context and initialize the delegate early,
      // if possible. If the root application context will be started after this
      // filter proxy, we'll have to resort to lazy initialization.
      WebApplicationContext wac = findWebApplicationContext();
      if (wac != null) {
        this.delegate = initDelegate(wac);
      }
    }
  }
}
```

```java
protected final String getFilterName() {
  return (this.filterConfig != null ? this.filterConfig.getFilterName() : this.beanName);
}
```

```java
protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
  Filter delegate = wac.getBean(getTargetBeanName(), Filter.class);
  if (isTargetFilterLifecycle()) {
    delegate.init(getFilterConfig());
  }
  return delegate;
}
```

## 控制 Spring Session 的启用和关闭
在项目开发过程中，本地开发环境，多数情况下是不需要依赖 redis 做 Session 共享的。一方面使用共享，会增加启动时间；另一方面，若是测试环境的 redis 连接不上，导致项目无法启动，影响开发时间。故需要有一个开关，用于配置当前环境（一般会有开发环境、测试环境、外测环境、线上环境）是否要开启 redis session 共享。

Session 共享开发如何实现？因为 Spring Session 共享功能的类均是由 Spring IOC 进行加载创建实例的，所以要根据项目中各个环境配置的开关来控制实例的创建。若使用 xml 配置，就不方便动态管理类实例创建，所以采用 Java Config 的方式来动态控制。

```java
@Configuration
public class RedisBeanLoadConfig {

    @Autowired
    private SystemConfig systemConfig;
    
    @Bean
    public JedisConnectionFactory jedisConnectionFactory() {
        JedisConnectionFactory jedisConnectionFactory = null;
    
        if (systemConfig.getRedisLoadSwitch()) {
    
            // JedisConnectionFactory
            jedisConnectionFactory = new JedisConnectionFactory(redisSentinelConfiguration,
                    jedisPoolConfig);
        }
        return jedisConnectionFactory;
    }
    
    @Bean
    public RedisHttpSessionConfiguration redisHttpSessionConfiguration(ApplicationContext ac) {
    
        RedisHttpSessionConfiguration redisHttpSessionConfiguration = null;
        if (systemConfig.getRedisLoadSwitch() && systemConfig.getSpringSessionSwitch()) {
    
            AnnotationConfigWebApplicationContext acwac = new AnnotationConfigWebApplicationContext();
            acwac.setParent(ac);
            acwac.register(RedisHttpSessionConfiguration.class);
            acwac.refresh();
            SessionRepositoryFilter sessionRepositoryFilter = acwac.getBeanFactory().getBean(
                    SessionRepositoryFilter.class);
    
            // register acwac 中的对象实例到 xwac
      XmlWebApplicationContext xwac = (XmlWebApplicationContext) ac;
      xwac.getBeanFactory().registerSingleton("springSessionRepositoryFilter",
                    sessionRepositoryFilter);
    }
    return redisHttpSessionConfiguration;
  }
}
```

- systemConfig.getRedisLoadSwitch() 用于控制 redis 连接池的实例创建。
- systemConfig.getSpringSessionSwitch() 用于控制是否开启 Spring Session ，若systemConfig.getRedisLoadSwitch()设置为false，则Spring Session 将强制关闭。

```java
public class SpringSessionFilterProxyInitializer extends AbstractHttpSessionApplicationInitializer {

    private SystemConfig systemConfig = new SystemConfig();
    
    private static final String SERVLET_CONTEXT_PREFIX = "org.springframework.web.servlet.FrameworkServlet.CONTEXT.";
    public static final String DEFAULT_FILTER_NAME = "springSessionRepositoryFilter";
    private static final String SPRING_DISPATCHER_SERVLET_NAME = "springDispatcher";
    
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        if (systemConfig != null && systemConfig.getRedisLoadSwitch()
                && systemConfig.getSpringSessionSwitch()) {
            DelegatingFilterProxy springSessionRepositoryFilter = new DelegatingFilterProxy(
                    DEFAULT_FILTER_NAME);
            String contextAttribute = getWebApplicationContextAttribute();
            if (contextAttribute != null) {
                springSessionRepositoryFilter.setContextAttribute(contextAttribute);
            }
            FilterRegistration.Dynamic dynamicFilter = servletContext.addFilter(
                    "DEFAULT_FILTER_NAME", springSessionRepositoryFilter);
            dynamicFilter.setAsyncSupported(true);
            dynamicFilter.addMappingForServletNames(super.getSessionDispatcherTypes(), false,
                    SPRING_DISPATCHER_SERVLET_NAME);
        }
    }
    
    private String getWebApplicationContextAttribute() {
        String dispatcherServletName = getDispatcherWebApplicationContextSuffix();
        if (dispatcherServletName == null) {
            return null;
        }
        return SERVLET_CONTEXT_PREFIX + dispatcherServletName;
    }

}
```

Spring Session 提供了一个实现 AbstractHttpSessionApplicationInitializer ，只需要重写相应的 onStartup() 方法，加入开关逻辑即可。此实现必须要求 **Servlet 3.0+** ，基于 ServletContainerInitializer 来实现的。
