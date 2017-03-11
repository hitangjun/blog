---
title: Spring Cloud Netflix--Zuul 三
date: 2017-03-11 09:55:18
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


### Uploading Files through Zuul
If you @EnableZuulProxy you can use the proxy paths to upload files and it should just work as long as the files are small. For large files there is an alternative path which bypasses the Spring DispatcherServlet (to avoid multipart processing) in "/zuul/*". I.e. if zuul.routes.customers=/customers/** then you can POST large files to "/zuul/customers/*". The servlet path is externalized via zuul.servletPath. Extremely large files will also require elevated timeout settings if the proxy route takes you through a Ribbon load balancer, e.g.

如果你使用 @EnableZuulProxy , 你可以使用代理路径上传文件, 对于小文件可以正常使用. 对于大文件有可选的路径”/zuul/“绕过Spring DispatcherServlet (避免处理multipart). 比如对于 zuul.routes.customers=/customers/* , 你可以使用 “/zuul/customers/*” 去上传大文件. Servlet路径通过 zuul.servletPath 指定. 如果使用Ribbon负载均衡器的代理路由, 在 处理非常大的文件时, 仍然需要提高超时配置. 比如:
<!--more-->
application.yml

	hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 60000
	ribbon:
	  ConnectTimeout: 3000
	  ReadTimeout: 60000

> Note that for streaming to work with large files, you need to use chunked encoding in the request (which some browsers do not do by default). E.g. on the command line:
> 
> 注意: 对于大文件的上传流, 你应该在请求中使用块编码. (有些浏览器默认不这么做). 比如在命令行中:


	$ curl -v -H "Transfer-Encoding: chunked" \
	    -F "file=@mylarge.iso" localhost:9999/zuul/simple/file


### Query String Encoding
When processing the incoming request, query params are decoded so they can be available for possible modifications in Zuul filters. They are then re-encoded when building the backend request in the route filters. The result can be different than the original input if it was encoded using Javascript’s encodeURIComponent() method for example. While this causes no issues in most cases, some web servers can be picky with the encoding of complex query string.

To force the original encoding of the query string, it is possible to pass a special flag to ZuulProperties so that the query string is taken as is with the HttpServletRequest::getQueryString method :

application.yml
 zuul:
  forceOriginalQueryStringEncoding: true
Note: This special flag only works with SimpleHostRoutingFilter and you loose the ability to easily override query parameters with RequestContext.getCurrentContext().setRequestQueryParams(someOverriddenParameters) since the query string is now fetched directly on the original HttpServletRequest.

### Plain Embedded Zuul
You can also run a Zuul server without the proxying, or switch on parts of the proxying platform selectively, if you use @EnableZuulServer (instead of @EnableZuulProxy). Any beans that you add to the application of type ZuulFilter will be installed automatically, as they are with @EnableZuulProxy, but without any of the proxy filters being added automatically.

你可以运行一个没有代理功能的Zuul服务, 或者有选择的开关部分代理功能, 如果你使用 @EnableZuulServer (替代 @EnableZuulProxy ). 你添加的任何 ZuulFilter 类型 实体类都会被自动加载, 和使用 @EnableZuulProxy 一样, 但不会自动加载任何代理过滤器.

In this case the routes into the Zuul server are still specified by configuring "zuul.routes.*", but there is no service discovery and no proxying, so the "serviceId" and "url" settings are ignored. For example:

在以下例子中, Zuul服务中的路由仍然是按照 “zuul.routes.*”指定, 但是没有服务发现和代理, 因此”serviceId”和”url”配置会被忽略. 比如:

application.yml

	 zuul:
	  routes:
	    api: /api/**

maps all paths in "/api/**" to the Zuul filter chain.

匹配所有的 “/api/**” 给Zuul过滤器链.

### Disable Zuul Filters
Zuul for Spring Cloud comes with a number of ZuulFilter beans enabled by default in both proxy and server mode. See the zuul filters package for the possible filters that are enabled. If you want to disable one, simply set zuul.<SimpleClassName>.<filterType>.disable=true. By convention, the package after filters is the Zuul filter type. For example to disable org.springframework.cloud.netflix.zuul.filters.post.SendResponseFilter set zuul.SendResponseFilter.post.disable=true.

在代理和服务模式下, 对于Spring Cloud, Zuul默认加入了一批 ZuulFilter 类. 查阅 the zuul filters package 去获取可能开启的过滤器. 如果你想关闭其中一个, 可以简单的设置 zuul...disable=true . 按照约定, 在 filter 后面的包是Zuul过滤器类. 比如关闭 org.springframework.cloud.netflix.zuul.filters.post.SendResponseFilter , 可设置zuul.SendResponseFilter.post.disable=true.


### Providing Hystrix Fallbacks For Routes
When a circuit for a given route in Zuul is tripped you can provide a fallback response by creating a bean of type ZuulFallbackProvider. Within this bean you need to specify the route ID the fallback is for and provide a ClientHttpResponse to return as a fallback. Here is a very simple ZuulFallbackProvider implementation.

	class MyFallbackProvider implements ZuulFallbackProvider {
	    @Override
	    public String getRoute() {
	        return "customers";
	    }
	
	    @Override
	    public ClientHttpResponse fallbackResponse() {
	        return new ClientHttpResponse() {
	            @Override
	            public HttpStatus getStatusCode() throws IOException {
	                return HttpStatus.OK;
	            }
	
	            @Override
	            public int getRawStatusCode() throws IOException {
	                return 200;
	            }
	
	            @Override
	            public String getStatusText() throws IOException {
	                return "OK";
	            }
	
	            @Override
	            public void close() {
	
	            }
	
	            @Override
	            public InputStream getBody() throws IOException {
	                return new ByteArrayInputStream("fallback".getBytes());
	            }
	
	            @Override
	            public HttpHeaders getHeaders() {
	                HttpHeaders headers = new HttpHeaders();
	                headers.setContentType(MediaType.APPLICATION_JSON);
	                return headers;
	            }
	        };
	    }
	}
And here is what the route configuration would look like.

	zuul:
	  routes:
	    customers: /customers/**
If you would like to provide a default fallback for all routes than you can create a bean of type ZuulFallbackProvider and have the getRoute method return * or null.

	class MyFallbackProvider implements ZuulFallbackProvider {
	    @Override
	    public String getRoute() {
	        return "*";
	    }
	
	    @Override
	    public ClientHttpResponse fallbackResponse() {
	        return new ClientHttpResponse() {
	            @Override
	            public HttpStatus getStatusCode() throws IOException {
	                return HttpStatus.OK;
	            }
	
	            @Override
	            public int getRawStatusCode() throws IOException {
	                return 200;
	            }
	
	            @Override
	            public String getStatusText() throws IOException {
	                return "OK";
	            }
	
	            @Override
	            public void close() {
	
	            }
	
	            @Override
	            public InputStream getBody() throws IOException {
	                return new ByteArrayInputStream("fallback".getBytes());
	            }
	
	            @Override
	            public HttpHeaders getHeaders() {
	                HttpHeaders headers = new HttpHeaders();
	                headers.setContentType(MediaType.APPLICATION_JSON);
	                return headers;
	            }
	        };
	    }
	}


### How Zuul Errors Work
If an exception is thrown during any portion of the Zuul filter lifecycle, the error filters are executed. The SendErrorFilter is only run if RequestContext.getThrowable() is not null. It then sets specific javax.servlet.error.* attributes in the request and forwards the request to the Spring Boot error page.

### Polyglot support with Sidecar 通过Sidecar进行多语言支持
Do you have non-jvm languages you want to take advantage of Eureka, Ribbon and Config Server? The Spring Cloud Netflix Sidecar was inspired by Netflix Prana. It includes a simple http api to get all of the instances (ie host and port) for a given service. You can also proxy service calls through an embedded Zuul proxy which gets its route entries from Eureka. The Spring Cloud Config Server can be accessed directly via host lookup or through the Zuul Proxy. The non-jvm app should implement a health check so the Sidecar can report to eureka if the app is up or down.

你是否有多语言的需要使用Eureka，Ribbon和Config server？ Spring Cloud Netflix Sidecar 受 Netflix Prana 启发。它引入了一个简单的HTTP API去获取所有服务实例的信息（比如host和port）。你也可以通过依赖Eureka的嵌入式Zuul代理器代理服务调用。Spring Cloud Config Server以通过host查找或Zuul代理直接访问。其他语言需要实现一个健康检查器，Sidecar才可以通知eureka这个额app是上线还是下线状态。

To include Sidecar in your project use the dependency with group org.springframework.cloud and artifact id spring-cloud-netflix-sidecar.

引入Sidecar需要org.springframework.cloud and artifact id spring-cloud-netflix-sidecar.

To enable the Sidecar, create a Spring Boot application with @EnableSidecar. This annotation includes @EnableCircuitBreaker, @EnableDiscoveryClient, and @EnableZuulProxy. Run the resulting application on the same host as the non-jvm application.

开启Sidecar, 需要创建一个包含 @EnableSidecar 的Springboot应用程序. 这个注解包括了 @EnableCircuitBreaker, @EnableDiscoveryClient 和 @EnableZuulProxy。Run the resulting application on the same host as the non-jvm application.

To configure the side car add sidecar.port and sidecar.health-uri to application.yml. The sidecar.port property is the port the non-jvm app is listening on. This is so the Sidecar can properly register the app with Eureka. The sidecar.health-uri is a uri accessible on the non-jvm app that mimicks a Spring Boot health indicator. It should return a json document like the following:

配置Sidecar, 添加 sidecar.port and sidecar.health-uri 到 application.yml 中. 属性 sidecar.port 配置非jvm应用正在监听的端口. 这样Sidecar能够注册应用到 Eureka. sidecar.health-uri 是一个非JVM应用程序提供模仿SpringBoot健康检查接口的可访问的uri. 它应该返回一个json文档类似如下:

	health-uri-document
	{
	  "status":"UP"
	}
Here is an example application.yml for a Sidecar application:

application.yml

	server:
	  port: 5678
	spring:
	  application:
	    name: sidecar

	sidecar:
	  port: 8000
	  health-uri: http://localhost:8000/health.json

The api for the DiscoveryClient.getInstances() method is /hosts/{serviceId}. Here is an example response for /hosts/customers that returns two instances on different hosts. This api is accessible to the non-jvm app (if the sidecar is on port 5678) at http://localhost:5678/hosts/{serviceId}.

	/hosts/customers
	[
	    {
	        "host": "myhost",
	        "port": 9000,
	        "uri": "http://myhost:9000",
	        "serviceId": "CUSTOMERS",
	        "secure": false
	    },
	    {
	        "host": "myhost2",
	        "port": 9000,
	        "uri": "http://myhost2:9000",
	        "serviceId": "CUSTOMERS",
	        "secure": false
	    }
	]
The Zuul proxy automatically adds routes for each service known in eureka to /<serviceId>, so the customers service is available at /customers. The Non-jvm app can access the customer service via http://localhost:5678/customers (assuming the sidecar is listening on port 5678).

Zuul会自动的为每一个eureka的服务添加路由映射为/，所以/customers可以访问到customers服务。非JVM的应用可以通过http://localhost:5678/customers(假设sidecar监听在5678)。

If the Config Server is registered with Eureka, non-jvm application can access it via the Zuul proxy. If the serviceId of the ConfigServer is configserver and the Sidecar is on port 5678, then it can be accessed at http://localhost:5678/configserver

如果Config Server注册到Eureka，非JVM的应用可以通过Zuul proxy访问。如果ConfigServer的serviceId是 configserver 而且Sidecar监听在5678端口上, 则它可以通过 http://localhost:5678/configserver 访问到.

Non-jvm app can take advantage of the Config Server’s ability to return YAML documents. For example, a call to http://sidecar.local.spring.io:5678/configserver/default-master.yml might result in a YAML document like the following

非JVM应用可以使用ConfigServer的功能返回YAML文档. 比如, 调用 http://sidecar.local.spring.io:5678/configserver/default-master.yml 可以返回如下文档:

	eureka:
	  client:
	    serviceUrl:
	      defaultZone: http://localhost:8761/eureka/
	  password: password
	info:
	  description: Spring Cloud Samples
	  url: https://github.com/spring-cloud-samples
