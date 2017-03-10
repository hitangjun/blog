---
title: Spring Cloud Netflix--Hystrix Clients
date: 2017-03-10 14:54:20
description: 
categories:
- Spring Cloud Netflix
tags:
- Circuit Breaker
- Hystrix 
toc: true
author: John Tang
comments:
original:
permalink: 
---


## Circuit Breaker: Hystrix Clients 断路器
Netflix has created a library called Hystrix that implements the circuit breaker pattern. In a microservice architecture it is common to have multiple layers of service calls.

Netflix 创建了一个库叫 [Hystrix](https://github.com/Netflix/Hystrix) ，它实现了 [circuit breaker pattern](http://martinfowler.com/bliki/CircuitBreaker.html)。 在microservice架构中通常有多个层的服务调用。
<!-- more -->
HystrixGraph

![](http://cloud.spring.io/spring-cloud-netflix/images/HystrixGraph.png)

*Figure 1. Microservice Graph*

A service failure in the lower level of services can cause cascading failure all the way up to the user. When calls to a particular service reach a certain threshold (20 failures in 5 seconds is the default in Hystrix), the circuit opens and the call is not made. In cases of error and an open circuit a fallback can be provided by the developer.

一个低水平的服务群中一个服务挂掉会给用户带来连锁故障的。当调用一个特定的服务达到一定阈值(在Hystrix里默认是5秒内20次故障)，断路由开启，并且调用不会成功。在断路器开启及故障的情况下，可以由开发人员进行回退。
 

HystrixFallback
![](http://cloud.spring.io/spring-cloud-netflix/images/HystrixFallback.png)
*Figure 2. Hystrix fallback prevents cascading failures*


Having an open circuit stops cascading failures and allows overwhelmed or failing services time to heal. The fallback can be another Hystrix protected call, static data or a sane empty value. Fallbacks may be chained so the first fallback makes some other business call which in turn falls back to static data.

通过断路器来避免连锁故障，从而可能拥有时间来修复过载和出现故障的服务。回调能被另一个Hystrix保护调用，静态数据或者合理的空数据。回调 可能被绑定，从而第一次回调会有一些其它的业务调用反馈才落回到静态数据。


### How to Include Hystrix 如何引入Hystrix
To include Hystrix in your project use the starter with group org.springframework.cloud and artifact id spring-cloud-starter-hystrix. See the Spring Cloud Project page for details on setting up your build system with the current Spring Cloud Release Train.

在你的项目中通过 org.springframework.cloud 和 spring-cloud-starter-hystrix 引入Hystrix。可以参照[Spring Cloud Project](http://projects.spring.io/spring-cloud/)的细节来设置你的构建系统。

Example boot app:

	@SpringBootApplication
	@EnableCircuitBreaker
	public class Application {
	
	    public static void main(String[] args) {
	        new SpringApplicationBuilder(Application.class).web(true).run(args);
	    }
	
	}
	
	@Component
	public class StoreIntegration {
	
	    @HystrixCommand(fallbackMethod = "defaultStores")
	    public Object getStores(Map<String, Object> parameters) {
	        //do stuff that might fail
	    }
	
	    public Object defaultStores(Map<String, Object> parameters) {
	        return /* something useful */;
	    }
	}


The @HystrixCommand is provided by a Netflix contrib library called "javanica". Spring Cloud automatically wraps Spring beans with that annotation in a proxy that is connected to the Hystrix circuit breaker. The circuit breaker calculates when to open and close the circuit, and what to do in case of a failure.

Netflix的普通发布库叫[Javanica](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica)，提供了@HystrixCommand注解。Spring Cloud使用注解自动适配spring bean使用代理去连接到Hystrix断路器。断路器计算何时打开和关闭断路，并决定在失败的情况下做什么。

To configure the @HystrixCommand you can use the commandProperties attribute with a list of @HystrixProperty annotations. See here for more details. See the Hystrix wiki for details on the properties available.

配置@HystrixCommand你可以使用commandProperties属性，它有@HystrixProperty的注解列表。通过[这里](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica#configuration)查看更多详情.访问[Hystrix wiki](https://github.com/Netflix/Hystrix/wiki/Configuration)查看更多可用的属性。

### Propagating the Security Context or using Spring Scopes 传播安全上下文或使用 Spring Scopes

If you want some thread local context to propagate into a @HystrixCommand the default declaration will not work because it executes the command in a thread pool (in case of timeouts). You can switch Hystrix to use the same thread as the caller using some configuration, or directly in the annotation, by asking it to use a different "Isolation Strategy". For example:

如果你想把本地线程上下文传播到@HystrixCommand，默认的声明将不可用因为它是在一个线程池中被启动的。你可以选择让Hystrix使用同一个线程，通过一些配置，或直接写在注解上，通过使用isolation strategy属性。例如：

	@HystrixCommand(fallbackMethod = "stubMyService",
	    commandProperties = {
	      @HystrixProperty(name="execution.isolation.strategy", value="SEMAPHORE")
	    }
	)
	...

The same thing applies if you are using @SessionScope or @RequestScope. You will know when you need to do this because of a runtime exception that says it can’t find the scoped context.

同样的方式适用于如果你用@SessionScope 或者 @RequestScope。你应该知道什么时候去做这件事因为有些运行时异常报找不到scoped上下文。

You also have the option to set the hystrix.shareSecurityContext property to true. Doing so will auto configure an Hystrix concurrency strategy plugin hook who will transfer the SecurityContext from your main thread to the one used by the Hystrix command. 

你还可以选择设置 hystrix.shareSecurityContext 属性为true。设置这个值会自动配置一个Hystrix并发策略会把securityContext从主线程传输到你使用的Hystrix command。

Hystrix does not allow multiple hystrix concurrency strategy to be registered so an extension mechanism is available by declaring your own HystrixConcurrencyStrategy as a Spring bean. Spring Cloud will lookup for your implementation within the Spring context and wrap it inside its own plugin.



### Health Indicator 健康指标

The state of the connected circuit breakers are also exposed in the /health endpoint of the calling application.

断路器的状态同样暴露在/health端点上。

	{
	    "hystrix": {
	        "openCircuitBreakers": [
	            "StoreIntegration::getStoresByLocationLink"
	        ],
	        "status": "CIRCUIT_OPEN"
	    },
	    "status": "UP"
	}

### Hystrix Metrics Stream ， Hystrix 指标流

To enable the Hystrix metrics stream include a dependency on spring-boot-starter-actuator. This will expose the /hystrix.stream as a management endpoint.

使用Hystrix metrics stream需要引入依赖 spring-boot-starter-actuator。这会暴露/hystrix.stream作为一个管理端点。

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

