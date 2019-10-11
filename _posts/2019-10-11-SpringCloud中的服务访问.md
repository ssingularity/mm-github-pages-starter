---
title: "SpringCloud中的服务访问"
date: 2019-10-11
categories:
  - Spring Cloud
tags:
  - Spring Cloud
  - Gateway
header:
  teaser: /assets/images/Spring-Cloud-Gateway.jpg
---
![image](/assets/images/Spring-Cloud-Gateway.jpg)

最近在对于一个以前写的会议室管理系统进行重构，微服务选型采用了Spring Cloud作为框架，其中使用spring-cloud-gateway作为统一网关替代了zuul，并细致了解了一下服务间调用所用到的一些东西，，在这里就来谈一谈在Spring Cloud框架下一些服务间调用的过程，例如Feign，Ribbon，Hystrix各有什么用，怎么配置，以及为什么要使用spring-cloud-gateway和怎么配置

## Feign、Ribbon和Hystrix
Hystrix是用来做熔断，它主要在目标服务不可用的时候快速失败，从而防止雪崩，Ribbon是用来做负载均衡的，它通过注册到Eureka的ServiceId来查找并负载均衡对应的服务，Feign用来做注解式服务调用，它简化了服务间调用的方式，并且把Hystrix和Ribbon包含了进来，所以说服务调用的流程是通过Feign开始的，然后Feign先开启Hystrix开始熔断监控以及服务保护，然后Ribbon找到对应的ip，最后开始调用底层的Http框架进行服务调用（Feign把整个服务调用都包含进来了，可以认为Feign有自己的Hystrix和Ribbon，所以只要把Feign导入进来，Hystrix和Ribbon就都有了）。

其中一些值得记录的配置方面的东西是：
- 可以使用 ribbon.MaxAutoRetries以及ribbon相关配置来实现重试效果（feign底层使用了ribbon），但同时需要加入spring-retry依赖。
- 这些流程（包括重试、熔断）都是作用在服务调用端的，服务端不care这些
- 如果要启用Hystrix就要配置feign.hystrix.enabled=true，如果有Retry，则需要注意Hystrix以及ribbon的超时配置，前者应该大于后者
- 使用hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds配置Hystrix超时，其中default代表一种hystrix默认command，hystrix默认使用default的配置，当然也可以声明配置自己其他的自定义command

## Spring-Cloud-Gateway
Spring Cloud Gateway 是 Spring Cloud 的一个全新项目，该项目是基于 Spring 5.0，Spring Boot 2.0 和 Project Reactor 等技术开发的网关，它旨在为微服务架构提供一种简单有效的统一的 API 路由管理方式。

相比之前我们使用的 Zuul（1.x） 它有哪些优势呢？其中最大的优势就是Zuul（1.x） 基于 Servlet，使用阻塞 API，它不支持任何长连接，如 WebSockets而Spring Cloud Gateway 使用非阻塞 API，支持 WebSockets，支持限流等新特性。

Spring Cloud Gateway的特征：
- 基于 Spring Framework 5，Project Reactor 和 Spring Boot 2.0
- 动态路由
- Predicates 和 Filters 作用于特定路由
- 集成 Hystrix 断路器
- 集成 Spring Cloud DiscoveryClient
- 易于编写的 Predicates 和 Filters
- 限流
- 路径重写

一个最简单的例子：
```
  spring:
  cloud:
    gateway:
      routes:
      - id: neo_route
        uri: http://www.ityouknow.com
        predicates:
        - Path=/spring-cloud
```
各个字段含义如下：
- id：我们自定义的路由 ID，保持唯一
- uri：目标服务地址
- predicates：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。除了Path，Spring Cloud Gateway还支持很多其他的字段，如Host，Method，Header，Cookie，After等等
- filters：过滤规则，主要用来做一下匹配加重写，本示例暂时没用
- 一个请求满足多个路由的谓词条件时，请求只会被首个成功匹配的路由转发

在实际的工作中，服务的相互调用都是依赖于服务中心提供的入口来使用，服务中心往往注册了很多服务，如果每个服务都需要单独配置的话，这将是一份很枯燥的工作。Spring Cloud Gateway 提供了一种默认转发的能力，只要将 Spring Cloud Gateway 注册到服务中心，Spring Cloud Gateway 默认就会代理服务中心的所有服务

示例如下：
```
  spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
```
其中lower-case-service-id字段将service-id转化为了小写，不然默认是大写，关于和服务注册中心连接的配置这里就不贴出来了

但有的时候这样的自动配置又过于笼统，真实的场景下需要我们针对每个微服务自定义化一些不同的操作，比如Retry，Fallback等，这时候就需要我们手动的配置，并且使用到Gateway的filters

示例如下：
```
spring:
  cloud:
    gateway:
      routes:
      - id: test
        uri: lb://test-service
        predicates:
        - Path=/test/**
        filters:
        - name: Hystrix
          args:
            name: fallbackcmd
            fallbackUri: forward:/fallback
        - name: Retry
          args:
            retries: 3
            series:
            - SERVER_ERROR
        - RewritePath=/test/(?<segment>.*), /$\{segment}

hystrix:
  command:
    fallbackcmd:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 5000
```
各个字段解释如下：
- uri字段中的lb://前缀是因为服务注册到了注册中心，所以可以直接通过服务名的方式来进行转发并且进行负载均衡
- Hystrix过滤器主要是开启Hystrix熔断功能，fallbackcmd在后面的hrixtrix.command.fallbackcmd定义并配置了对应的超时时间，fallbackUri是熔断了以后的回调方法，也即访问gateway对应的uri，这里需要自己去定义
- Retry过滤器主要是开启了重试功能，series字段定义了在什么情况下进行重试，默认是5xx
- RewritePath过滤器是在请求转发的时候进行Uri的重写，不然会原样转发到对应服务，造成Uri不匹配