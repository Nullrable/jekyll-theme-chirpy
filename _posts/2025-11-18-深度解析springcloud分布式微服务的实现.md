---
title: 深度解析springcloud分布式微服务的实现
author: nhsoft.lsd
date: 2025-11-14
categories: [Java]
pin: false
---

# 分布式系统
微服务就是原来臃肿的项目拆分为多个模块互不关联。如：按照子服务拆分、数据库、接口，依次往下就更加细粒度，当然运维也就越来越难受了。

分布式则是偏向与机器将诺大的系统划分为多个模块部署在不同服务器上。

微服务和分布式就是作用的“目标不一样”。

# 微服务与Spring Cloud
微服务是一种概念，spring-cloud是微服务的实现。

微服务也不一定必须使用cloud来实现，只是微服务中有许多问题，如：负载均衡、服务注册与发现、路由等等。

而cloud则是将这些处理问题的技术整合了。

# Spring-Cloud 组件

## Eureka

Eureka是Netifix的子模块之一，Eureka有2个组件，一个EurekaServer 实现中间层服务器的负载均衡和故障转移，一个EurekaClient它使得与server交互变得简单。

Spring-Cloud封装了Netifix公司开发的Eureka模块来实现服务注册和发现。

通过Eureka的客户端 Eureka Server维持心跳连接，维护可以更方便监控各个微服务的运行。

角色关系图

![img_1.png](../assets/img/nhsoft_lsd/img_1.png)

Eureka使用

客户端

```
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

server:
port: 4001
eureka:
client:
serviceUrl:
defaultZone: http://localhost:3000/eureka/ #eureka服务端提供的注册地址 参考服务端配置的这个路径
instance:
instance-id: admin-1 #此实例注册到eureka服务端的唯一的实例ID
prefer-ip-address: true #是否显示IP地址
leaseRenewalIntervalInSeconds: 10 #eureka客户需要多长时间发送心跳给eureka服务器，表明它仍然活着,默认为30 秒 (与下面配置的单位都是秒)
leaseExpirationDurationInSeconds: 30 #Eureka服务器在接收到实例的最后一次发出的心跳后，需要等待多久才可以将此实例删除，默认为90秒

spring:
application:
name: server-admin #此实例注册到eureka服务端的name

```

服务端

```
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-
netflix-eureka-server</artifactId>
</dependency>
yml文件声明
server:
port: 3000
eureka:
server:
enable-self-preservation: false #关闭自我保护机制
eviction-interval-timer-in-ms: 4000 #设置清理间隔
（单位：毫秒 默认是60*1000）
instance:
hostname: localhost
client:
registerWithEureka: false #不把自己作为一个客户端注册到自己身上
fetchRegistry: false #不需要从服务端获取注册信息
（因为在这里自己就是服务端，而且已经禁用自己注册了）
serviceUrl:
defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka
在SpringBoot 启动项目中加入注解:@EnableEurekaServer
就可以启动项目了，访问对应地址就可以看到界面。
```

## Eureka 集群

服务启动后Eureka Server会向其他服务server 同步，当消费者要调用服务提供者，则向服务注册中心获取服务提供者的地址，然后将提供者的地址缓存到本地，下次调用时候直接从本地缓存中获取

yml 服务端
```
server:
port: 3000
eureka:
server:
enable-self-preservation: false #关闭自我保护机制
eviction-interval-timer-in-ms: 4000 #设置清理间隔
（单位：毫秒 默认是60*1000）
instance:
hostname: eureka3000.com
client:
registerWithEureka: false #不把自己作为一个客户端
注册到自己身上
fetchRegistry: false #不需要从服务端获取注册信息
（因为在这里自己就是服务端，而且已经禁用自己注册了）
serviceUrl:
defaultZone: http://eureka3001.com:3001/eureka,
http://eureka3002.com:3002/eureka
(这里不注册自己，注册到其他服务上面以为会同步。)
```

yml 客户端

```
server:
port: 4001
eureka:
client:
serviceUrl:
defaultZone:http://localhost:3000/eureka/,http://
eureka3001.com:3001/eureka,http://eureka3002.com:3 002
/eureka #eureka服务端提供的注册地址 参考服务端配置的这个路径
instance:
instance-id: admin-1 #此实例注册到eureka服务端的唯一的实例ID
prefer-ip-address: true #是否显示IP地址
leaseRenewalIntervalInSeconds: 10 #eureka客户需要多长时间发
送心跳给eureka服务器，表明它仍然活着,默认为30 秒 (与下面配置的单位都是秒)
leaseExpirationDurationInSeconds: 30 #Eureka服务器在
接收到实例的最后一次发出的心跳后，需要等待多久才可以将此实例删除，默认为90秒

spring:
application:
name: server-admin #此实例注册到eureka服务端的name
```

## CAP定理

```
C：Consistency 一致性
A：Availability 可用性
P：Partition tolerance 分区容错性
```

这三个指标不能同时达到
**Partition tolerance**

分区容错性，大多数分布式系统都部署在多个子网络。每一个网络是一个区。区间的通信是可能失败的如一个在本地，一个在外地，他们之间是无法通信的。分布式系统在设计的时候必须要考虑这种情况。

**Consistency**

一致性，写操作后的读取，必须返回该值。如：服务器A1和服务器A2，现在发起操作将A1中V0改为V1，用户去读取的时候读到服务器A1得到V1，如果读到A2服务器但是服务器

还是V0，读到的数据就不对，这就不满足一致性。

所以让A2返回的数据也对，的让A1给A2发送一条消息，把A2的V0变为V1，这时候不管从哪里读取都是修改后的数据。

**Availability**

可用性就是用户只要给出请求就必须回应，不管是本地服务器还是外地服务器只要接收到就必须做出回应，不管数据是否是最新必须做出回应，负责就不是可用性。

**C与A矛盾**

一致性和可用性不能同时成立，存在分区容错性，通信可能失败。

如果保证一致性，A1在写操作时，A2的读写只能被锁定，只有等数据同步了才能读写，在锁定期间是不能读写的就不符合可用性。

如果保持可用性，那么A2就不会被锁定，所以一致性就不能成立。

综上 无法做到一致性和可用性，所以系统在设计的时候就只能选其一。

## Eureka与Zookeeper

Zookeeper遵循的是CP原则保持了一致性，所以在master节点因为网络故障与剩余“跟随者”接点失去联系时会重新选举“领导者”，选取“领导者”大概会持续30-120s的时间，且选举的时候整个zookeeper是不可用的。导致在选举的时候注册服务瘫痪。

Eureka在设计的时候遵循AP可用性。Eureka各个接点是公平的，没有主从之分，down掉几个几点也没问题，其他接点依然可以支持注册，只要有一台Eureka在，注册就可以用，只不过查询到的数据可能不是最新的。Eureka有自我保护机制，如果15分钟之内超过85%接点都没有正常心跳，那么Eureka认为客户端与注册中心出现故障，此时情况可能是

Eureka不在从注册列表移除因为长时间没有瘦到心跳而过期的服务。

Eureka仍然能够接收注册和查询，但不会同步到其他接点。

当网络稳定后，当前的 实例注册信息会更新到其他接点。

### Ribbon

rebbon主要提供客户端的负载均衡，提供了一套完善的客户端的配置。Rebbin会自动帮助你基于某种规则（如：简单的轮询，随机链接等）。

服务端的负载均衡是一个url通过一个代理服务器，然后通过代理服务器（策略：轮询，随机 ，权重等等），反向代理到你的服务器。

客户端负载均衡是通过一个请求在客户端已经声明了要调用那个服务，然后通过具体算法来完成负载均衡。

### Ribbon使用

引入依赖，Eureka以及把Ribbon集成在里面。

使用Ribbon只有在RestTemplate上面加入@LoadBalanced注解。

### Feign负载均衡

feign是一个声明式的webService客户端，使用feign会让编写webService更简单，就是定义一个接口加上注解。

feign是为了编写java http客户端更加简单，在Ribbon+RestTemplate此基础上进一步封装，简化了使用Spring Cloud Ribbon时，自动封装服务调用客户端的开发量。

### Feign使用

```
引入依赖
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
在启动类上加@EnableFeignClients
然后在接口上加@FeignClient("SERVER-POWER")注解其中参数就是服务的名字。
```

### Feign集成Ribbon

利用Ribbon维护服务列表信息，融合了Ribbon的负载均衡配置，与Ribbon不同的是Feign只需要定义服务绑定接口以声明的方式，实现简答的服务调用。

## hystrix断路器
是一种用于处理分布式系统延迟和容错的开源库。在分布式系统中许多依赖不可避免的会调用失败，比如超时、异常等，断路器保证出错不会导致整体服务失败，避免级联故障。

断路器其实就是一种开关设置，类似保险丝，像调用方返回一个符合预期的、可处理的备选响应，而不是长时间等待或者抛出无法处理的异常，保证服务调用方线程不会被长时间 不必要占用，从而避免了在分布式系统中蔓延，乃至雪崩。

微服务中 client->微服务A->微服务B->微服务C->微服务D,其中微服务B异常了，所有请求微服务A的请求都会卡在B这里，就会导致线程一直累积在这里，那么其他微服务就没有可用线程，导致整个服务器雪崩。

针对这方案有 服务限流、超时监控、服务熔断、服务降级

### 降级 超时

降级就是服务响应过长 ，或者不可用了，就是服务调用不了了，我们不能把错误信息返回出来，或者长时间卡在哪里，所以要准备一个策略当发生这种问题我们直接调用这个方法快速返回这个请求，不让他一直卡在那。

要在调用方做降级（要不然那个微服务都down掉了在做降级就没有意义）。

引入hystrix依赖

```
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
在启动类上加入@EnableHystrix 或者@EnableCircuitBreaker。
@RequestMapping("/feignPower.do")
@HystrixCommand(fallbackMethod = "fallbackMethod")
public Object feignPower(String name){
return powerServiceClient.power();
}
fallbackMethod：
public Object fallbackMethod(String name){
System.out.println(name);
return R.error("降级信息");
}
这里的降级信息具体内容根据业务需求来，比如返回一个默认的查询信息等等。
hystrix有超时监听，当你请求超过1秒 就会超时，这个是可以配置的
```

这里的降级信息具体内容根据业务需求来，比如返回一个默认的查询信息等等。

hystrix有超时监听，当你请求超过1秒 就会超时，这个是可以配置的

### 降级什么用

第一他可以监听服务有没有超时。第二报错了他这里直接截断了没有让请求一直卡在这个。

其实降级，当你系统迎来高并发的时候，这时候发现系统马上承载不了这个大的并发 ，可以先关闭一些不重要 的微服务（就是在降级方法返回一个比较友好的信息）把资源让出来给主服务，其实就是整体资源不够用了，忍痛关闭某些服务，待过渡后再打开。

### 熔断限流

熔断就像生活中的跳闸，比如电路故障了，为了防止事故扩大，这里切断你的电源以免意外发生。当一个微服务调用多次，hystrix就会采取熔断 机制，不在继续调用你的方法，会默认短路，5秒后试探性的先关闭熔断机制，如果在这时候失败一次会直接调用降级方法，一定程度避免雪崩，

限流，限制某个微服务使用量，如果线程占用超过了，超过的就会直接降级该次调用。

### Feign整合hystrix

```
feign默认支持hystrix，需要在yml配置中打开。
feign:
hystrix:
enabled: true

降级方法
@FeignClient(value = "SERVER-POWER", fallback = PowerServiceFallBack.class)
public interface PowerServiceClient {

@RequestMapping("/power.do")
public Object power(@RequestParam("name") String name);
}

在feign客户端的注解上 有个属性叫fallback 然后指向一个类 PowerServiceClient
@Component
public class PowerServiceFallBack implements PowerServiceClient {
@Override
public Object power(String name) {
return R.error("测试降级");
}
}
```

## Zuul 网关
zuul包含了对请求的路由和过滤两个主要功能

路由是将外部请求转发到具体的微服务实例上。是实现统一入口基础而过滤器功能负责对请求的处理过程干预，是实现请求校验等功能。

Zuul与Eureka进行整合，将zuul注册在Eureka服务治理下，同时从Eureka获取其他服务信息。（zuul分服务最终还是注册在Eureka上）

路由
```
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
最后要注册在Eureka上所以需要引入eureka依赖
YML
server:
port: 9000
eureka:
client:
serviceUrl:
defaultZone: http://localhost:3000/eureka/ #eureka服务端提供的注册地址 参考服务端配置的这个路径
instance:
instance-id: zuul-0 #此实例注册到eureka服务端的唯一的实例ID
prefer-ip-address: true #是否显示IP地址
leaseRenewalIntervalInSeconds: 10 #eureka客户需要多长时间发送心跳给eureka服务器，表明它仍然活着,默认为30 秒 (与下面配置的单位都是秒)
leaseExpirationDurationInSeconds: 30 #Eureka服务器在接收到实例的最后一次发出的心跳后，需要等待多久才可以将此实例删除，默认为90秒

spring:
application:
name: zuul #此实例注册到eureka服务端的name
启动类 @EnableZuulProxy

在实际开发当中我们肯定不会/server-power这样通过微服务调用，
可能只要一个/power就好了
zuul:
routes:
mypower:
serviceId: server-power
path: /power/**
myorder:
serviceId: server-order
path: /order/**
注意/**代表是所有层级 /* 是代表一层。
一般我们会禁用服务名调用
ignored-services：server-order 这样就不能通过此服务名调用，
不过这个配置如果一个一个通微服务名字设置太复杂
一般禁用服务名 ignored-services：“*”
有时候要考虑到接口调用需要一定的规范，比如调用微服务URL需要前缀/api，可以加上一个prefix
prefix：/api 在加上strip-prefix: false /api前缀是不会出现在路由中
zuul:
prefix: /api
ignored-services: "*"
stripPrefix: false
routes:
product:
serviceId: server-product
path: /product/**
order:
serviceId: server-order
path: /order/**
```

### 过滤器

过滤器(filter)是zuul的核心组件，zuul大部分功能是通过过滤器实现的，zuul中定义了4种标准过滤器类型，这些过滤器类型对应与请求的生命周期，

PRE：这种过滤器在请求路由前被调用，可利用过滤器进行身份验证，记录请求微服务的调试信息等。

ROUTING：这种过滤器将请求路由到微服务，这种过滤器用于构建发送给微服务请求，并使用 Apache HttpClient或Netfix Ribbon请求微服务。

POST：这种过滤器在路由微服务后执行，可用来相应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端。

ERROR：在其他阶段发送错误时执行过滤器

### 继承ZuulFilter

```java

@Component
public class LogFilter extends ZuulFilter {
@Override
public String filterType() {
return FilterConstants.PRE_TYPE;
}

@Override
public int filterOrder() {
return FilterConstants.PRE_DECORATION_FILTER_ORDER+1;
}

@Override
public boolean shouldFilter() {
return true;
}

@Override
public Object run() throws ZuulException {
RequestContext ctx = RequestContext.getCurrentContext();
//被代理到的微服务
String proxy = (String)ctx.get("proxy");
//请求的地址
String requestURI = (String)ctx.get("requestURI");
//zuul路由后的url
System.out.println(proxy+"/"+requestURI);
HttpServletRequest request = ctx.getRequest();
String loginCookie = CookieUtil.getLoginCookie(request);
ctx.addZuulRequestHeader("login_key",loginCookie);
return null;
}
}

```
由此可知道自定义zuul Filter要实现以下几个方法。

filterType：返回过滤器类型，有pre、route、post、erro等几种取值

filterOrder：返回一个int值指定过滤器的顺序，不同过滤器允许返回相同数字。

shouldFilter：返回一个boolean判断过滤器是否执行，true执行，false不执行。

run：过滤器的具体实现。

Spting-Cloud默认为zuul编写并开启一些过滤器。如果要禁用部分过滤器，只需在application.yml里设置zuul…disable=true，例如zuul.LogFilter.pre.disable=true

zuul也整合了了hystrix和ribbon的， 提供降级回退，继承FallbackProvider 类 然后重写里面的方法。

转载自并发编程网 – ifeve.com本文链接地址: 深度解析springcloud分布式微服务的实现

![weixin.png](/assets/img/nhsoft_lsd/weixin.png)

<div style="text-align: center;">公众号名称：怪味Coding</div>
<div style="text-align: center;">微信扫码关注或搜索公众号名称</div>
