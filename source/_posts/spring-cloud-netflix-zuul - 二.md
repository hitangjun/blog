---
title: Spring Cloud Netflix--Zuul 二
date: 2017-03-11 08:55:18
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


The zuul.routes entries actually bind to an object of type ZuulProperties. If you look at the properties of that object you will see that it also has a "retryable" flag. Set that flag to "true" to have the Ribbon client automatically retry failed requests (and if you need to you can modify the parameters of the retry operations using the Ribbon client configuration).

zuul.routes 实际上绑定到类型为 ZuulProperties 的对象上. 如果你查看这个对象你会发现一个叫”retryable”的字段, 设置为”true”会使Ribbon客户端自动在失败时重试(如果你需要修改重试参数, 可以使用Ribbon client configuration)

<!-- more-->
The X-Forwarded-Host header is added to the forwarded requests by default. To turn it off set zuul.addProxyHeaders = false. The prefix path is stripped by default, and the request to the backend picks up a header "X-Forwarded-Prefix" ("/myusers" in the examples above).

X-Forwarder-Host请求头默认添加到转发请求中。设置zuul.addProxyHeaders=false禁用它。路径前缀默认被删除， 
到后台服务的请求会添加一个 “X-Forwarded-Prefix”(“/myusers”在上面的例子中)。

An application with @EnableZuulProxy could act as a standalone server if you set a default route ("/"), for example zuul.route.home: / would route all traffic (i.e. "/**") to the "home" service.

一个@EnableZuulProxy的应用可以作为单机使用，如果你设置了一个默认路由（”/”），例如zuul.route.home: / 会把所有的请求（”/**”）转到home服务。

If more fine-grained ignoring is needed, you can specify specific patterns to ignore. These patterns are evaluated at the start of the route location process, which means prefixes should be included in the pattern to warrant a match. Ignored patterns span all services and supersede any other route specification.

如果需要更细粒度的忽略配置，你可以指定特殊的表达式来配置忽略规则.这些表达式从route location的开始进行匹配，意味着前缀应该被包括在匹配表达式中. 忽略表达式影响所有服务和取代任何路由的特殊配置.

application.yml

	 zuul:
	  ignoredPatterns: /**/admin/**
	  routes:
	    users: /myusers/**

This means that all calls such as "/myusers/101" will be forwarded to "/101" on the "users" service. But calls including "/admin/" will not resolve.

这个的意思是所有请求, 比如”/myusers/101”的请求会跳转到”users”服务的”/101”, 但包含”/admin/”的请求将不被处理.

WARNING
If you need your routes to have their order preserved you need to use a YAML file as the ordering will be lost using a properties file. For example:

application.yml

	 zuul:
	  routes:
	    users:
	      path: /myusers/**
	    legacy:
	      path: /**

If you were to use a properties file, the legacy path may end up in front of the users path rendering the users path unreachable.

### Zuul Http Client
The default HTTP client used by zuul is now backed by the Apache HTTP Client instead of the deprecated Ribbon RestClient. To use RestClient or to use the okhttp3.OkHttpClient set ribbon.restclient.enabled=true or ribbon.okhttp.enabled=true respectively.

默认的zull的Http clietn现在是Apach HTTP Client,替代了已过期的Ribbon RestClient。想使用RestClient或使用okhttp3.OKHttpClient,可以设置ribbon.restclient.enable=true或者ribbon.okhttp.enable=true。

### Cookies and Sensitive Headers
It’s OK to share headers between services in the same system, but you probably don’t want sensitive headers leaking downstream into external servers. You can specify a list of ignored headers as part of the route configuration. Cookies play a special role because they have well-defined semantics in browsers, and they are always to be treated as sensitive. If the consumer of your proxy is a browser, then cookies for downstream services also cause problems for the user because they all get jumbled up (all downstream services look like they come from the same place).

在同一个系统的多个服务之间中分享headers是可以的，但是你可能不想把一些敏感headers泄露到下游服务器。你可以指定一批忽略的headers列表在路由配置中。Cookies扮演了一个特殊的角色, 因为他们很好的在浏览器中定义, 而且他们总是被认为是敏感的. 如果代理的客户端是浏览器, 则对于下游服务来说对用户, cookies会引起问题, 因为他们都混在一起。(所有下游服务看起来认为他们来自同一个地方)。


If you are careful with the design of your services, for example if only one of the downstream services sets cookies, then you might be able to let them flow from the backend all the way up to the caller. Also, if your proxy sets cookies and all your back end services are part of the same system, it can be natural to simply share them (and for instance use Spring Session to link them up to some shared state). Other than that, any cookies that get set by downstream services are likely to be not very useful to the caller, so it is recommended that you make (at least) "Set-Cookie" and "Cookie" into sensitive headers for routes that are not part of your domain. Even for routes that are part of your domain, try to think carefully about what it means before allowing cookies to flow between them and the proxy.

如果你对于你的服务设计很细心，比如，如果只有一个下游的服务设置了cookies，你可能会让它从后端服务一直追溯到前端调用者，如果你的代理设置了cookies而且所有你的后端服务都是同一系统的一部分，它可以很自然的共享（比如使用spring session去联系一些共享状态）。除此之外，任何下游服务设置的cookies可以能不会对前端调用者产生作用。所以建议对不属于你的域名的部分在routes里将 “Set-Cookie”和“Cookie”添加到敏感headers。 即使是属于你的域名的路由, 尝试仔细思考在允许cookies流传在它们和代理之间意味着什么。


The sensitive headers can be configured as a comma-separated list per route, e.g.

每个路由中的敏感头部信息配置按照逗号分隔, 例如:

application.yml

	 zuul:
	  routes:
	    users:
	      path: /myusers/**
	      sensitiveHeaders: Cookie,Set-Cookie,Authorization
	      url: https://downstream

> NOTE
> this is the default value for sensitiveHeaders, so you don’t need to set it unless you want it to be different. N.B. this is new in Spring Cloud Netflix 1.1 (in 1.0 the user had no control over headers and all cookies flow in both directions).
> 
> 注意: 这是sensitiveHeaders 的默认值, 你无需设置除非你需要不同的配置. 注意. 这是Spring Cloud Netflix 1.1的新功能(在1.0中, 用户无法直接控制请求头和所有cookies).



The sensitiveHeaders are a blacklist and the default is not empty, so to make Zuul send all headers (except the "ignored" ones) you would have to explicitly set it to the empty list. This is necessary if you want to pass cookie or authorization headers to your back end. Example:

application.yml

	 zuul:
	  routes:
	    users:
	      path: /myusers/**
	      sensitiveHeaders:
	      url: https://downstream

Sensitive headers can also be set globally by setting zuul.sensitiveHeaders. If sensitiveHeaders is set on a route, this will override the global sensitiveHeaders setting.

敏感headers也支持全局设置 zuul.sensitiveHeaders. 如果在单个路由中设置 sensitiveHeaders 会覆盖全局 sensitiveHeaders 设置.

### Ignored Headers
In addition to the per-route sensitive headers, you can set a global value for zuul.ignoredHeaders for values that should be discarded (both request and response) during interactions with downstream services. By default these are empty, if Spring Security is not on the classpath, and otherwise they are initialized to a set of well-known "security" headers (e.g. involving caching) as specified by Spring Security. The assumption in this case is that the downstream services might add these headers too, and we want the values from the proxy. To not discard these well known security headers in case Spring Security is on the classpath you can set zuul.ignoreSecurityHeaders to false. This can be useful if you disabled the HTTP Security response headers in Spring Security and want the values provided by downstream services

除了每个route敏感头以外, 你可以设置一个全局的 zuul.ignoredHeaders 在下游相互调用间去丢弃这些值(包括请求和响应). 如果没有将Spring Security 添加到运行路径中, 他们默认是空的, 否则他们会被Spring Secuity初始化一批安全头(例如 缓存相关). 在这种情况下, 假设下游服务也可能添加这些头信息, 我希望从代理获取值.

### The Routes Endpoint
If you are using @EnableZuulProxy with tha Spring Boot Actuator you will enable (by default) an additional endpoint, available via HTTP as /routes. A GET to this endpoint will return a list of the mapped routes. A POST will force a refresh of the existing routes (e.g. in case there have been changes in the service catalog).

如果你使用 @EnableZuulProxy 同时引入了Spring Boot Actuator, 你将默认增加一个endpoint, 提供http服务的 /routes. 一个GET请求将返回路由匹配列表. 一个POST请求将强制刷新已存在的路由.(比如, 在服务catalog变化的场景中)

> NOTE
> the routes should respond automatically to changes in the service catalog, but the POST to /routes is a way to force the change to happen immediately.
> 
> 注意：路由列表应该自动应答服务登记变化, 但是POST是一种强制立即更新的方案.

### Strangulation Patterns and Local Forwards 窒息模式和本地跳转
A common pattern when migrating an existing application or API is to "strangle" old endpoints, slowly replacing them with different implementations. The Zuul proxy is a useful tool for this because you can use it to handle all traffic from clients of the old endpoints, but redirect some of the requests to new ones.

Example configuration:

一个常见的迁移旧应用或者旧接口的方式，就是逐步的替换它的实现。 Zuul代理是一种很有用的工具, 因为你可以使用这种方式处理所有客户端到旧接口的请求. 只是重定向了一些请求到新的接口.

实例配置：

application.yml
	
	 zuul:
	  routes:
	    first:
	      path: /first/**
	      url: http://first.example.com
	    second:
	      path: /second/**
	      url: forward:/second
	    third:
	      path: /third/**
	      url: forward:/3rd
	    legacy:
	      path: /**
	      url: http://legacy.example.com

In this example we are strangling the "legacy" app which is mapped to all requests that do not match one of the other patterns. Paths in /first/** have been extracted into a new service with an external URL. And paths in /second/** are forwarded so they can be handled locally, e.g. with a normal Spring @RequestMapping. Paths in /third/** are also forwarded, but with a different prefix (i.e. /third/foo is forwarded to /3rd/foo).

在这个例子中，我们替换了 “legacy” ，它映射到所有的请求，但是没有匹配到其他任何一个请求。路径 /first/* 指向了一个额外的URL. 并且路径 /second/* 是一个本地跳转. 比如, 带有Spring注解的 @RequestMapping . 路径 /third/** 也是一个本地跳转, 但是属于一个不同的前缀. (比如 /third/foo 跳转到 /3rd/foo )。

> NOTE
> The ignored patterns aren’t completely ignored, they just aren’t handled by the proxy (so they are also effectively forwarded locally).
> 
> 注意：忽略表达式并不是完全的忽略请求, 只是配置这个代理不处理这些请求(所以他们也是跳转执行本地处理)。


