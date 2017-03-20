---
title: Spring Cloud Netflix--Eureka Clients
date: 2017-03-09 17:21:20
description: 
categories:
- Spring Cloud Netflix
tags:
- Eureka Clients
toc: true
author: John Tang
comments:
original:
permalink: 
---

# Spring Cloud Netflix  

(1.3.0.BUILD-SNAPSHOT)

This project provides Netflix OSS integrations for Spring Boot apps through autoconfiguration and binding to the Spring Environment and other Spring programming model idioms. With a few simple annotations you can quickly enable and configure the common patterns inside your application and build large distributed systems with battle-tested Netflix components. The patterns provided include Service Discovery (Eureka), Circuit Breaker (Hystrix), Intelligent Routing (Zuul) and Client Side Load Balancing (Ribbon).


该项目通过自动配置和绑定到Spring环境和其他Spring编程模型惯例,为Spring Boot应用程序提供Netflix OSS集成。 通过几个简单的注解，您可以快速启用和配置应用程序中的常见功能模块，并使用久经考验的Netflix组件构建大型分布式系统。 提供的功能模块包括服务发现（Eureka），断路器（Hystrix），智能路由（Zuul）和客户端负载均衡（Ribbon）。
<!-- more -->

## Service Discovery: Eureka Clients 服务发现:Eureka客户端
Service Discovery is one of the key tenets of a microservice based architecture. Trying to hand configure each client or some form of convention can be very difficult to do and can be very brittle. Eureka is the Netflix Service Discovery Server and Client. The server can be configured and deployed to be highly available, with each server replicating state about the registered services to the others.

服务发现是microservice基础架构的关键原则之一。尝试亲手配置每个客户端或某种格式的约定可以说是非常困难的和非常脆弱的。Eureka是Netflix服务发现的一种服务和客户端。这种服务是可以被高可用性配置的和部署,并且在注册的服务当中，每个服务的状态可以互相复制给彼此。


### How to Include Eureka Client 如何使用Eureka Client
To include Eureka Client in your project use the starter with group org.springframework.cloud and artifact id spring-cloud-starter-eureka. See the Spring Cloud Project page for details on setting up your build system with the current Spring Cloud Release Train.


通过在项目pom.xml中引入group为org.springframework.cloud 和 artifact id为spring-cloud-starter-eureka可以快速引用Eureka Client。当您需要使用当前Spring Cloud 发布版本配置您的项目时，请参考Spring Cloud 项目页面以获取更详细信息。


### Registering with Eureka 注册到Eureka
When a client registers with Eureka, it provides meta-data about itself such as host and port, health indicator URL, home page etc. Eureka receives heartbeat messages from each instance belonging to a service. If the heartbeat fails over a configurable timetable, the instance is normally removed from the registry.

当一个客户端注册到Eureka,它提供关于自己的元数据（诸如主机和端口，健康指标URL，首页等）。Eureka通过一个服务从各个实例接收心跳信息。如果心跳接收失败超过配置的时间，实例将会正常从注册表里面移除。

Example eureka client:

``` java
	@Configuration
	@ComponentScan
	@EnableAutoConfiguration
	@EnableEurekaClient
	@RestController
	public class Application {

	    @RequestMapping("/")
	    public String home() {
		return "Hello world";
	    }

	    public static void main(String[] args) {
		new SpringApplicationBuilder(Application.class).web(true).run(args);
	    }

	}
``` 
(i.e. utterly normal Spring Boot app). In this example we use @EnableEurekaClient explicitly, but with only Eureka available you could also use @EnableDiscoveryClient. Configuration is required to locate the Eureka server. Example:

在这个例子里我们使用 @EnableEurekaClient 来声明, 但只有使 Eureka 生效还得 使用 @EnableDiscoveryClient。 配置要求 定位Eureka服务端。 例如:


application.yml

    eureka:
	  client:
	    serviceUrl:
	      defaultZone: http://localhost:8761/eureka/

where "defaultZone" is a magic string fallback value that provides the service URL for any client that doesn’t express a preference (i.e. it’s a useful default).

"defaultZone"是一个神奇的字符串回退值，它提供了服务的URL给任何客户端，而这不意味优先级。 (i.e. 是一个常用的默认值).


The default application name (service ID), virtual host and non-secure port, taken from the Environment, are ${spring.application.name}, ${spring.application.name} and ${server.port} respectively.

默认应用名（服务ID），虚拟主机和非安全端口, 分别从环境的 ${spring.application.name}, ${spring.application.name} 和 ${server.port} 获取。

@EnableEurekaClient makes the app into both a Eureka "instance" (i.e. it registers itself) and a "client" (i.e. it can query the registry to locate other services). The instance behaviour is driven by eureka.instance.* configuration keys, but the defaults will be fine if you ensure that your application has a spring.application.name (this is the default for the Eureka service ID, or VIP).

@EnableEurekaClient 使Eureka既是一个实例（向自身注册），同时又是客户端（它能通过查找注册表来定位其它服务）。实例的行为由eureka.instance.*的配置项来决定,但是你最好确保你的spring.application.name有个默认值。（这是Eureka的默认ID或VIP）。

See [EurekaInstanceConfigBean](http://github.com/spring-cloud/spring-cloud-netflix/tree/master/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaInstanceConfigBean.java) and [EurekaClientConfigBean](http://github.com/spring-cloud/spring-cloud-netflix/tree/master/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaClientConfigBean.java) for more details of the configurable options.

更多配置项的细节请看[EurekaInstanceConfigBean](http://github.com/spring-cloud/spring-cloud-netflix/tree/master/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaInstanceConfigBean.java) 和 [EurekaClientConfigBean](http://github.com/spring-cloud/spring-cloud-netflix/tree/master/spring-cloud-netflix-eureka-client/src/main/java/org/springframework/cloud/netflix/eureka/EurekaClientConfigBean.java) 

###Authenticating with the Eureka Server 对Eureka服务的身份验证
HTTP basic authentication will be automatically added to your eureka client if one of the eureka.client.serviceUrl.defaultZone URLs has credentials embedded in it (curl style, like http://user:password@localhost:8761/eureka). For more complex needs you can create a @Bean of type DiscoveryClientOptionalArgs and inject ClientFilter instances into it, all of which will be applied to the calls from the client to the server.

如果其中一个eureka.client.serviceUrl.defaultZone的url已经把凭证嵌入到它里面，那么HTTP基本的身份验证将会被自动添加到你的eureka客户端(curl风格,如 http://user:password@localhost:8761/eureka)。 对于更复杂的需求，您可以创建一个带“@Bean”注解的“DiscoveryClientOptionalArgs”类型并且为它注入“ClientFilter”实例。


> #### NOTE
> Because of a limitation in Eureka it isn’t possible to support per-server basic auth credentials, so only the first set that are found will be used.
> 
> 由于Eureka的一个限制是不可能支持每个服务器基本授权认证,所以只被发现的第一组会被使用。
> 


### Status Page and Health Indicator 健康指标和状态页面
The status page and health indicators for a Eureka instance default to "/info" and "/health" respectively, which are the default locations of useful endpoints in a Spring Boot Actuator application. You need to change these, even for an Actuator application if you use a non-default context path or servlet path (e.g. server.servletPath=/foo) or management endpoint path (e.g. management.contextPath=/admin). Example:

健康指标和状态页面分别对应一个Eureka实例的"/health"和"/info"，是在一个Spring Boot Actuator应用默认的配置位置中很有用的一个点。即便是一个Actuator应用，如果你使用非默认的上下文路径或者servlet路径（如server.servletPath=/foo）或管理端点的路径（如management.contextPath=/admin），你都需要做出相应的改变。例如：

application.yml

	eureka:
	  instance:
	    statusPageUrlPath: ${management.context-path}/info
	    healthCheckUrlPath: ${management.context-path}/health

These links show up in the metadata that is consumed by clients, and used in some scenarios to decide whether to send requests to your application, so it’s helpful if they are accurate.

这些链接呈现出被客户端所消耗的基础数据,并且如果这些信息准确的话，当需要决定是否发送请求给你应用程序的某些情况下，它们是有用的。

### Registering a Secure Application  注册一个安全应用
If your app wants to be contacted over HTTPS you can set two flags in the EurekaInstanceConfig, viz eureka.instance.[nonSecurePortEnabled,securePortEnabled]=[false,true] respectively. This will make Eureka publish instance information showing an explicit preference for secure communication. The Spring Cloud DiscoveryClient will always return an https://…​; URI for a service configured this way, and the Eureka (native) instance information will have a secure health check URL.

如果你的应用想使用HTTPS，你需要设置两个标记，分别是EurekaInstanceConfig, viz eureka.instance.[nonSecurePortEnabled,securePortEnabled]=[false,true]。这将使Eureka推送实例的信息展示一个显式的安全通信。Spring Cloud的DiscoveryClient将总是通过这种方式返回一个服务的配置的URI（https://…​;）, 并且Eureka实例（native）的信息将有一个安全的健康检查的URL。


Because of the way Eureka works internally, it will still publish a non-secure URL for status and home page unless you also override those explicitly. You can use placeholders to configure the eureka instance urls, e.g.

因为Eureka的内部工作方式，它将继续推送一个非安全的URL的状态和主页，除非你已覆盖那些声明。你可以使用占位符或者配置eureka实例的url。 例如：

application.yml

	eureka:
	  instance:
	    statusPageUrl: https://${eureka.hostname}/info
	    healthCheckUrl: https://${eureka.hostname}/health
	    homePageUrl: https://${eureka.hostname}/
(Note that ${eureka.hostname} is a native placeholder only available in later versions of Eureka. You could achieve the same thing with Spring placeholders as well, e.g. using ${eureka.instance.hostName}.)

(请注意 ${eureka.hostname} 是一个原生占位符只可用在以后的版本的Eureka.你也可以使用Spring的占位符做同样的事情, 例如使用 ${eureka.instance.hostName}.)


>#### NOTE
> If your app is running behind a proxy, and the SSL termination is in the proxy (e.g. if you run in Cloud Foundry or other platforms as a service) then you will need to ensure that the proxy "forwarded" headers are intercepted and handled by the application. An embedded Tomcat container in a Spring Boot app does this automatically if it has explicit configuration for the 'X-Forwarded-\*` headers. A sign that you got this wrong will be that the links rendered by your app to itself will be wrong (the wrong host, port or protocol).
> 
> 如果你的应用在慢于一个代理启动运行，并且SSL终端在代理里面（如：如果你的应用作为一个服务在Cloud Foundry或其它平台上跑的话），那么你将要确保应用能够拦截和处理代理转发的头信息。如果它明确配置有'X-Forwarded-\*`这类头信息的情况下，在一个Spring Boot应用里面内嵌的Tomcat容器自动处理的。出现这个错误的原因，是因为你的应用程序提供的链接本身弄错了。（错误的主机，端口，协议）。


### Eureka’s Health Checks  Eureka 健康检查
By default, Eureka uses the client heartbeat to determine if a client is up. Unless specified otherwise the Discovery Client will not propagate the current health check status of the application per the Spring Boot Actuator. Which means that after successful registration Eureka will always announce that the application is in 'UP' state. This behaviour can be altered by enabling Eureka health checks, which results in propagating application status to Eureka. As a consequence every other application won’t be sending traffic to application in state other then 'UP'.

默认情况下,Eureka使用客户端心跳来确定一个客户端是否活着。除非另有指定Discovery Client不会传播当前Spring Boot Actuator的应用性能的健康检查状态。也就是说在成功注册Eureka后总会宣布的应用程序在“UP”的状态。这种发送应用状态给Eureka的行为将触发Eureka的健康检查。因此其他每个应用程序在其他状态下不会给应用程序发送通信然后才‘UP’。

application.yml

	eureka:
	  client:
	    healthcheck:
	      enabled: true
> WARNING
> eureka.client.healthcheck.enabled=true should only be set in application.yml. Setting the value in bootstrap.yml will cause undesirable side effects like registering in eureka with an UNKNOWN status.
> 
> 注意
> eureka.client.healthcheck.enabled=true 应该仅在application.yml中配置。在 bootstrap.yml 中配置将会导致无法预料的影响，如注册到Eureka时状态未知。


If you require more control over the health checks, you may consider implementing your own com.netflix.appinfo.HealthCheckHandler.

如果你需要更多超出健康检查的监控，你可以考虑实现自己的com.netflix.appinfo.HealthCheckHandler.


### Eureka Metadata for Instances and Clients  Eureka给客户端和实例的元数据
It’s worth spending a bit of time understanding how the Eureka metadata works, so you can use it in a way that makes sense in your platform. There is standard metadata for things like hostname, IP address, port numbers, status page and health check. These are published in the service registry and used by clients to contact the services in a straightforward way. Additional metadata can be added to the instance registration in the eureka.instance.metadataMap, and this will be accessible in the remote clients, but in general will not change the behaviour of the client, unless it is made aware of the meaning of the metadata. There are a couple of special cases described below where Spring Cloud already assigns meaning to the metadata map.

值得花一点时间了解Eureka元数据是如何工作的,从而可以在你的平台以某种有意义的方式使用它。标准的元数据，如主机名、IP地址、端口号、状态页面和健康检查。这些元数据在服务注册和客户端联系服务端时以一种简单的方式被推送出去。额外的元数据可以被添加到实例注册在eureka.instance.metadataMap里面,并且这都是在远程客户端可访问到的,但通常不会改变客户的行为,除非它是识别到元数据的含义。有一些特殊的情况：Spring Cloud已经分配元数据映射的含义。


### Changing the Eureka Instance ID 修改Eureka实例ID
A vanilla Netflix Eureka instance is registered with an ID that is equal to its host name (i.e. only one service per host). Spring Cloud Eureka provides a sensible default that looks like this: ${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id:${server.port}}}. For example myhost:myappname:8080.

Netflix Eureka实例是使用ID，如主机名（即只有一个服务/主机）进行注册的。Spring Cloud Eureka提供了合理的默认值，看起来像这样：${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id:${server.port}}}。例如` Myhost：myappname：8080 `。

Using Spring Cloud you can override this by providing a unique identifier in eureka.instance.instanceId. For example:

使用Spring Cloud 您可以通过eureka.instance.instanceId提供一个惟一的标识符覆盖它:

application.yml

	eureka:
	  instance:
	    instanceId: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}

With this metadata, and multiple service instances deployed on localhost, the random value will kick in there to make the instance unique. In Cloudfoundry the vcap.application.instance_id will be populated automatically in a Spring Boot application, so the random value will not be needed.

根据这种元数据，并且多个实例部署在localhost，随机值可以确保实例的唯一。但是在Cloudfoundry中，vcap.application.instance_id将被自动赋值在一个Spring Boot应用程序中，因此随机值将不再被需要。

### Using the EurekaClient 使用EurekaClient
Once you have an app that is @EnableDiscoveryClient (or @EnableEurekaClient) you can use it to discover service instances from the Eureka Server. One way to do that is to use the native com.netflix.discovery.EurekaClient (as opposed to the Spring Cloud DiscoveryClient), e.g.

当你有一个应用配置了 @EnableDiscoveryClient (或 @EnableEurekaClient) 你可以使用它从 Eureka Server发现服务实例。其中一个方法是使用原生的 com.netflix.discovery.EurekaClient (而不是 Spring Cloud的 DiscoveryClient), 例如：
		
	@Autowired
	private EurekaClient discoveryClient;
	
	public String serviceUrl() {
	    InstanceInfo instance = discoveryClient.getNextServerFromEureka("STORES", false);
	    return instance.getHomePageUrl();
	}

> TIP
> Don’t use the EurekaClient in @PostConstruct method or in a @Scheduled method (or anywhere where the ApplicationContext might not be started yet). It is initialized in a SmartLifecycle (with phase=0) so the earliest you can rely on it being available is in another SmartLifecycle with higher phase.
> 
> 不要在@PostConstruct方法或@Scheduled方法使用EurekaClient（或任何ApplicationContext还没被启动的地方）。 它初始化 SmartLifecycle (在 phase=0下的情况下) ，所以你最先可以在另一个SmartLifecycle的更高阶段依赖它生效。
>

### Alternatives to the native Netflix EurekaClient  代替原生的Netflix EurekaClient
You don’t have to use the raw Netflix EurekaClient and usually it is more convenient to use it behind a wrapper of some sort. Spring Cloud has support for Feign (a REST client builder) and also Spring RestTemplate using the logical Eureka service identifiers (VIPs) instead of physical URLs. To configure Ribbon with a fixed list of physical servers you can simply set <client>.ribbon.listOfServers to a comma-separated list of physical addresses (or hostnames), where <client> is the ID of the client.

你不一定要使用原始Netflix EurekaClient，并且，通常使用它背后的某种形式的封装是更方便的。 Spring Cloud已经支持Feign（一个REST客户端构建）并且Spring RestTemplate 也使用逻辑Eureka服务标识符(VIP)代替物理的URL。配置一个固定物理服务器的列表的Ribbon，你可以对<client>是客户端的ID，用一个逗号分隔物理地址（或主机名）列表来简单地设置<client>.ribbon.listOfServers。

You can also use the org.springframework.cloud.client.discovery.DiscoveryClient which provides a simple API for discovery clients that is not specific to Netflix, e.g.

你也可以使用org.springframework.cloud.client.discovery.DiscoveryClient提供一个简单的API给不确定的Netflix发现客户端。

	@Autowired
	private DiscoveryClient discoveryClient;
	
	public String serviceUrl() {
	    List<ServiceInstance> list = discoveryClient.getInstances("STORES");
	    if (list != null && list.size() > 0 ) {
	        return list.get(0).getUri();
	    }
	    return null;
	}


### Why is it so Slow to Register a Service? 为什么注册一个服务这么慢?

Being an instance also involves a periodic heartbeat to the registry (via the client’s serviceUrl) with default duration 30 seconds. A service is not available for discovery by clients until the instance, the server and the client all have the same metadata in their local cache (so it could take 3 heartbeats). You can change the period using eureka.instance.leaseRenewalIntervalInSeconds and this will speed up the process of getting clients connected to other services. In production it’s probably better to stick with the default because there are some computations internally in the server that make assumptions about the lease renewal period.

作为一个还包括一个默认持续30秒的周期心跳的实例向注册中心进行注册(通过客户的serviceUrl)，直到实例、服务端和客户端全都拥有相同的元数据在它们的缓存里面（这可能还需要3次心跳），一个服务对于客户端的discovery不可用的。你可以修改eureka.instance.leaseRenewalIntervalInSeconds并且这将加快这一进程的客户端连接到其他服务。在实际生产中坚持默认可能是更好的，因为有一些服务器内部计算来预估恢复时间。

### Zones
If you have deployed Eureka clients to multiple zones than you may prefer that those clients leverage services within the same zone before trying services in another zone. To do this you need to configure your Eureka clients correctly.

First, you need to make sure you have Eureka servers deployed to each zone and that they are peers of each other. See the section on zones and regions for more information.

Next you need to tell Eureka which zone your service is in. You can do this using the metadataMap property. For example if service 1 is deployed to both zone 1 and zone 2 you would need to set the following Eureka properties in service 1

Service 1 in Zone 1

	eureka.instance.metadataMap.zone = zone1
	eureka.client.preferSameZoneEureka = true
	Service 1 in Zone 2

	eureka.instance.metadataMap.zone = zone2
	eureka.client.preferSameZoneEureka = true

