---
title: Session、cookie、token概述
date: 2017-08-17 22:45:45
tags:
    - Session
    - Cookie
    - token
categories:
    - 计算机网络
---

我们知道http协议是一个无状态的协议，也就是不具备有记忆功能。但是在现在的网站大多具备软件的功能，因此就需要保持用户的状态，保存状态的技术主要有Session技术、Cookie技术。
## Session

Session技术是通过服务端来保存用户状态的。在Java的web容器中，都实现了HttpSession的接口。通过调用HttpServletRequest.getSession方法可以创建一个HttpSession。创建的同时，会生成一个SessionId，并返回到客户端。之后客户端的请求都会带有这个SessionId，服务器通过读取SessionId在服务器中获取对应的Session。

默认的Session会创建在服务器的内存中。在可扩展的服务当中，这样可能会带来一些问题：

- 存储大量的Session会占用大量的堆空间，可能会导致垃圾回收，使性能变差。
- 多个服务器之间要同步Session会使得Session保存了N份。

Session的删除时间是在Session超时或服务器关闭、停止时。超时是指如果一段时间没有受到Session对应的客户端的请求，并且这个时间超过了服务器设置的Session有效的最大时间，那么Session就会被删除。

Session最常用的方法就是 getAttribute() 方法和 setAttribute() 方法。


### Session共享
现在的互联网网站大多有多台服务器，每台服务器对外提供的服务是等价的，但是不同服务器上肯定是不同的web容器。因此如果Session继续存储在web容器的内存中的话，就会出现用户访问A服务器时生成了SessionId和Session对象，但是继续访问B服务器时SessionId在B的web容器中找不到对应的Session。因此就需要Session共享来保持Session的一致性。常见的Session共享方法有如下几种：

- iphash,把特定ip发送给特定主机，就不存在session这个问题了，因为1个用户对应1台主机。但是某时刻当来自某个IP地址的请求特别多，那么将导致某台负载服务器的压力可能非常大，而其他负载服务器却空闲的不均衡情况，这就违背了我们负载均衡的初衷。
- 搭建redis集群或者memcached集群，用集群自带的同步方法来帮我们在不同的主机中同步session，这样就相当于把原来的一份session变成了N分session（有点浪费，哈哈），session的同步就依赖于NoSql集群的同步了。
- 不使用session，换作cookie。但是秉承着防御性编程的原则，我们不能相信用户输入，因为cookie可能被禁用，甚至篡改。
- 单独设置一个session服务器，负载服务器得到一个sessionid过后，去session服务器获得会话状态，然后根据状态来响应用户请求，如果会话状态为空，则在session服务器中设置一个会话状态，然后返回给用户一个sessionid。

我们采用单独设置Session服务器的方法来实现Session共享。

### Spring Session与Redis实现Session共享
在Spring Boot项目中，可以使用Spring Session代替传统的HttpSession，并将Session单独的存储在Redis服务器（集群）中，可以达到高可用、可扩展、负载均衡的目的。
Spring Session对HTTP的支持是通过一个ServletFilter来实现的。具体的实现可以看[这里](http://www.infoq.com/cn/articles/Next-Generation-Session-Management-with-Spring-Session)。
具体的配置较为简单，步骤如下：
#### Maven依赖
```
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session</artifactId>
    <version>1.0.2.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-redis</artifactId>
</dependency>
```
#### 配置Spring Session Servlet Filter
如果是Spring Boot项目，只需一个注解@EnableRedisHttpSession即可，如下：
```
@SpringBootApplication
@EnableRedisHttpSession
public class ExampleApplication {

    public static void main(String[] args) {
        SpringApplication.run(ExampleApplication.class, args);
    }
}
```
如果是其他Spring框架，则需要手动配置web.xml：
```
<filter>
    <filter-name>springSessionRepositoryFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
    <filter-name>springSessionRepositoryFilter</filter-name>
    <url-pattern>/*</url-pattern>
        <dispatcher>REQUEST</dispatcher>
        <dispatcher>ERROR</dispatcher>
</filter-mapping>
```
#### Spring Session 到 Redis 的连接
Spring Boot中：
```
spring.redis.host=localhost
spring.redis.password=secret
spring.redis.port=6379
```
其他：
```
<bean class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration"/>
<bean class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
    <property name="hostName" value="localhost" />
    <property name="password" value="your-password" />
    <property name="port" value="6379" />
    <property name="database" value="10" />
</bean>
```

## Cookie
Cookie是HTTP用来保存状态的另一种技术，这种技术是客户端的解决方案。
Cookie就是由服务器发给客户端的特殊信息，而这些信息以文本文件的方式存放在客户端，然后客户端每次向服务器发送请求的时候都会带上这些特殊的信息。

说得更具体一些：当用户使用浏览器访问一个支持Cookie的网站的时候，用户会提供包括用户名在内的个人信息并且提交至服务器；接着，服务器在向客户端回传相应的超文本的同时也会发回这些个人信息，当然这些信息并不是存放在HTTP响应体（Response Body）中的，而是存放于HTTP响应头（Response Header）；当客户端浏览器接收到来自服务器的响应之后，浏览器会将这些信息存放在一个统一的位置，对于Windows操作系统而言，我们可以从： [系统盘]:\Documents and Settings\[用户名]\Cookies目录中找到存储的Cookie；自此，客户端再向服务器发送请求的时候，都会把相应的Cookie再次发回至服务器。而这次，Cookie信息则存放在HTTP请求头（Request Header）了。

有了Cookie这样的技术实现，服务器在接收到来自客户端浏览器的请求之后，就能够通过分析存放于请求头的Cookie得到客户端特有的信息，从而动态生成与该客户端相对应的内容。通常，我们可以从很多网站的登录界面中看到“请记住我”这样的选项，如果你勾选了它之后再登录，那么在下一次访问该网站的时候就不需要进行重复而繁琐的登录动作了，而这个功能就是通过Cookie实现的。

cookie和session的方案虽然分别属于客户端和服务端，但是服务端的session的实现对客户端的cookie有依赖关系的，上面讲到服务端执行session机制时候会生成session的id值，这个id值会发送给客户端，客户端每次请求都会把这个id值放到http请求的头部发送给服务端，而这个id值在客户端会保存下来，保存的容器就是cookie，因此当我们完全禁掉浏览器的cookie的时候，服务端的session也会不能正常使用。另一种方式是通过url中附带sessionid来实现。

## token
token在web中的含义是“令牌”，通常用来处理用户认证。
基于Token的身份验证的过程如下:
1. 用户通过用户名和密码发送请求。
2. 程序验证。
3. 程序返回一个签名的token 给客户端。
4. 客户端储存token,并且每次用于每次发送请求。
5. 服务端验证token并返回数据。
通常可以作为token认证的可以是sessionId或者是服务器下发的一个随机数值，token存储在cookie中。