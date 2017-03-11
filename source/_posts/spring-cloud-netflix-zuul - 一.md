---
title: Spring Cloud Netflix--Zuul 一
date: 2017-03-10 17:55:18
description: 
categories:
- Spring Cloud Netflix
tags:
- Zuul
toc: true
author: John Tang
comments:
original:
permalink: 
---

## Router and Filter: Zuul
Routing in an integral part of a microservice architecture. For example, / may be mapped to your web application, /api/users is mapped to the user service and /api/shop is mapped to the shop service. Zuul is a JVM based router and server side load balancer by Netflix.

路由是微服务架构的不可或缺的一部分。例如：/ 可能映射到你应用主页，/api/users映射到用户服务，/api/shop映射到购物服务。Zuul。Zuul是Netflix出品的一个基于JVM路由和服务端的负载均衡器。

<!-- more -->
Netflix uses Zuul for the following:

- Authentication
- Insights
- Stress Testing
- Canary Testing
- Dynamic Routing
- Service Migration
- Load Shedding
- Security
- Static Response handling
- Active/Active traffic management

Zuul’s rule engine allows rules and filters to be written in essentially any JVM language, with built in support for Java and Groovy.

Zuul的规则和过滤器允许使用各种基于JVM的语言，支持基于Java和Groovy。

> NOTE
> The configuration property zuul.max.host.connections has been replaced by two new properties, zuul.host.maxTotalConnections and zuul.host.maxPerRouteConnections which default to 200 and 20 respectively.
> 注意:zuul.max.host.connections已经被两个新的属性替代：zuul.host.maxTotalConnections 和 zuul.host.maxPerRouteConnections，默认分别为200和20.
> .
> NOTE
> Default Hystrix isolation pattern (ExecutionIsolationStrategy) for all routes is SEMAPHORE. zuul.ribbonIsolationStrategy can be changed to THREAD if this isolation pattern is preferred.
> 注意：默认所有routes的Hystrix隔离模式（ExecutionIsolationStrategy）是SEMAPHORE zuul.ribbonIsolationStrategy可以改为THREAD,如果这个隔离模式更好。

### How to Include Zuul
To include Zuul in your project use the starter with group org.springframework.cloud and artifact id spring-cloud-starter-zuul. See the Spring Cloud Project page for details on setting up your build system with the current Spring Cloud Release Train.


### Embedded Zuul Reverse Proxy
Spring Cloud has created an embedded Zuul proxy to ease the development of a very common use case where a UI application wants to proxy calls to one or more back end services. This feature is useful for a user interface to proxy to the backend services it requires, avoiding the need to manage CORS and authentication concerns independently for all the backends.

当一个UI应用想要代理调用一个或者多个后台服务的时候，Sping cloud创建了一个嵌入的Zuul proxy很方便的开发一个简单的案例。这个功能对于代理前端需要访问的后端服务非常有用，避免了所有后端服务需要关心管理CORS和认证的问题.

To enable it, annotate a Spring Boot main class with @EnableZuulProxy, and this forwards local calls to the appropriate service. By convention, a service with the ID "users", will receive requests from the proxy located at /users (with the prefix stripped). The proxy uses Ribbon to locate an instance to forward to via discovery, and all requests are executed in a hystrix command, so failures will show up in Hystrix metrics, and once the circuit is open the proxy will not try to contact the service.

在Spring Boot主函数上通过注解 @EnableZuulProxy 来开启, 这样可以让本地的请求转发到适当的服务. 按照约定, 一个ID为”users”的服务会收到 /users 请求路径的代理请求(前缀会被剥离). Zuul使用Ribbon定位服务注册中的实例, 并且所有的请求都在hystrix的command中执行, 所以失败信息将会展现在Hystrix metrics中, 并且一旦断路器打开, 代理请求将不会尝试去链接服务.


> NOTE
> the Zuul starter does not include a discovery client, so for routes based on service IDs you need to provide one of those on the classpath as well (e.g. Eureka is one choice).
> 
> 注意：Zuul starter没有包含服务发现的客户端, 所以对于路由你需要在classpath中提供一个根据service IDs做服务发现的服务.(例如, eureka是一个不错的选择)
> 

To skip having a service automatically added, set zuul.ignored-services to a list of service id patterns. If a service matches a pattern that is ignored, but also included in the explicitly configured routes map, then it will be unignored. Example:

去忽略一个自动添加的服务，可以在服务ID表达式列表中设置 zuul.ignored-services。如果一个服务匹配到了要忽略的列表, 但是它也明确的配置在路由列表中, 将不会被忽略, 例如:

application.yml

	 zuul:
	  ignoredServices: '*'
	  routes:
	    users: /myusers/**

In this example, all services are ignored except "users".

在这个例子中，所有的服务都会被忽略，除了“users”。

To augment or change the proxy routes, you can add external configuration like the following:

增加或改变代理路由规则, 你可以添加类似下面的外部配置:

application.yml

	 zuul:
	  routes:
	    users: /myusers/**

This means that http calls to "/myusers" get forwarded to the "users" service (for example "/myusers/101" is forwarded to "/101").

这表示，HTTP调用 “/myusers” 会转到 “user” 服务（例如：”/myusers/101”跳转到”/101”）。

To get more fine-grained control over a route you can specify the path and the serviceId independently:

为了更细粒度的控制一个路由, 你可以独立指定配置路径和服务ID:

application.yml

	 zuul:
	  routes:
	    users:
	      path: /myusers/**
	      serviceId: users_service

This means that http calls to "/myusers" get forwarded to the "users_service" service. The route has to have a "path" which can be specified as an ant-style pattern, so "/myusers/*" only matches one level, but "/myusers/**" matches hierarchically.

这表示，HTTP调用 “/myuser”会跳转到”users_servie”服务。路由必须配置一个可以被指定为”ant风格”的”path”，所以“/myusers/”只能匹配一个层级, 但"/myusers/*"可以匹配多级.（附注：[Ant path 匹配原则](http://blog.csdn.net/chenqipc/article/details/53289721)）

The location of the backend can be specified as either a "serviceId" (for a service from discovery) or a "url" (for a physical location), e.g.

后端的配置既可以是”serviceId”(对于服务发现中的服务), 也可以是”url”(物理地址), 例如:

application.yml

	 zuul:
	  routes:
	    users:
	      path: /myusers/**
	      url: http://example.com/users_service

These simple url-routes don’t get executed as a HystrixCommand nor can you loadbalance multiple URLs with Ribbon. To achieve this, specify a service-route and configure a Ribbon client for the serviceId (this currently requires disabling Eureka support in Ribbon: see above for more information), e.g.

url-routes的方式不会执行 HystrixCommand 也不会通过Ribbon负载多个URLS。要实现这些，需给这个serviceid指定一个service-route并配置一个Ribbon client（这个必须在Ribbon中禁用Eureka: see above for more information）。

application.yml

	zuul:
	  routes:
	    users:
	      path: /myusers/**
	      serviceId: users

	ribbon:
	  eureka:
	    enabled: false
	
	users:
	  ribbon:
	    listOfServers: example.com,google.com

You can provide convention between serviceId and routes using regexmapper. It uses regular expression named groups to extract variables from serviceId and inject them into a route pattern.

你可以使用regexmapper提供serviceId和routes之间的绑定. 它使用正则表达式组来从serviceId提取变量, 然后注入到路由表达式中.

ApplicationConfiguration.java

	@Bean
	public PatternServiceRouteMapper serviceRouteMapper() {
	    return new PatternServiceRouteMapper(
	        "(?<name>^.+)-(?<version>v.+$)",
	        "${version}/${name}");
	}

This means that a serviceId "myusers-v1" will be mapped to route “/v1/myusers/**”. Any regular expression is accepted but all named groups must be present in both servicePattern and routePattern. If servicePattern does not match a serviceId, the default behavior is used. In the example above, a serviceId "myusers" will be mapped to route "/myusers/**" (no version detected) This feature is disable by default and only applies to discovered services.

这表示serviceId “myusers-v1” 将会被映射到 /v1/myusers/ .任何正则表达式都可以，但是所有的命名组都必须在servicePattern和routePattern中存在。如果servicePattern没有匹配到一个serviceId，默认的行为会被启用。在上面的例子中，serviceId”myusers”将会映射到”/myusers/“(没有发现版本)这个特性默认是禁用的，而且只用于发现的服务。

To add a prefix to all mappings, set zuul.prefix to a value, such as /api. The proxy prefix is stripped from the request before the request is forwarded by default (switch this behaviour off with zuul.stripPrefix=false). You can also switch off the stripping of the service-specific prefix from individual routes, e.g.

给所有映射添加前缀，可以设置 zuul.prefix 一个值，比如/api。这个前缀默认会删除，在请求跳转之前。（通过 zuul.stripPrefix=false 可以关闭这个功能）。你也可以在单个服务中关闭这个功能, 例如:

application.yml

	 zuul:
	  routes:
	    users:
	      path: /myusers/**
	      stripPrefix: false

> NOTE
> zuul.stripPrefix only applies to the prefix set in zuul.prefix. It does have any effect on prefixes defined within a given route’s path.
> 
> zuul.stripPrefix只使用于使用了zuul.prefix配置情况下。在一个定义好了的 route’s path中不会有任何影响。

In this example, requests to "/myusers/101" will be forwarded to "/myusers/101" on the "users" service.

在这个例子中，”users”service的请求”/myusers/101”将会跳转到”/myusers/101”。
