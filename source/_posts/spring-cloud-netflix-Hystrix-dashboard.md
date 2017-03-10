---
title: Spring Cloud Netflix--Hystrix Dashboard
date: 2017-03-10 14:54:20
description: 
categories:
- Spring Cloud Netflix
tags:
- Circuit Breaker
- Hystrix 
- Dashboard
toc: true
author: John Tang
comments:
original:
permalink: 
---

## Circuit Breaker: Hystrix Dashboard Hystrix仪表盘

One of the main benefits of Hystrix is the set of metrics it gathers about each HystrixCommand. The Hystrix Dashboard displays the health of each circuit breaker in an efficient manner.

Hystrix的主要好处就是她收集了关于每个HystrixCommand的指标。Hystrix仪表盘用一种高效的方式展示了断路器的健康数据。 
<!-- more -->

Hystrix
![](http://cloud.spring.io/spring-cloud-netflix/images/Hystrix.png)
*Figure 3. Hystrix Dashboard*

### How to Include Hystrix Dashboard  如何引入Hystrix仪表盘
To include the Hystrix Dashboard in your project use the starter with group org.springframework.cloud and artifact id spring-cloud-starter-hystrix-dashboard. See the Spring Cloud Project page for details on setting up your build system with the current Spring Cloud Release Train.

在你的项目中通过 org.springframework.cloud 和 spring-cloud-starter-hystrix-dashboard 引入Hystrix dashboard。可以参照[Spring Cloud Project](http://projects.spring.io/spring-cloud/)的细节来设置你的构建系统。

To run the Hystrix Dashboard annotate your Spring Boot main class with @EnableHystrixDashboard. You then visit /hystrix and point the dashboard to an individual instances /hystrix.stream endpoint in a Hystrix client application.

在Spring boot main class上加@EnableHystrixDashboard可以运行Hystrix仪表盘，然后可以访问/hystrix并把仪表盘指向一个个体实例/hystrix.stream端点在一个应用中。

### Turbine
Looking at an individual instances Hystrix data is not very useful in terms of the overall health of the system. Turbine is an application that aggregates all of the relevant /hystrix.stream endpoints into a combined /turbine.stream for use in the Hystrix Dashboard. Individual instances are located via Eureka. Running Turbine is as simple as annotating your main class with the @EnableTurbine annotation (e.g. using spring-cloud-starter-turbine to set up the classpath). All of the documented configuration properties from the Turbine 1 wiki apply. The only difference is that the turbine.instanceUrlSuffix does not need the port prepended as this is handled automatically unless turbine.instanceInsertPort=false.

看一个实例Hystrix数据对于整个系统的健康不是很有用. Turbine 是一个应用程序,该应用程序汇集了所有相关的/hystrix.stream端点到 /turbine.stream用于Hystrix仪表板。运行turbine使用@EnableTurbine注解你的主类，使用spring-cloud-starter-turbine这个jar。配置请参考 the Turbine 1 wiki 唯一的区别是turbine.instanceUrlSuffix不需要端口号前缀,因为这是自动处理,除非turbine.instanceInsertPort=false。

> NOTE
> By default, Turbine looks for the /hystrix.stream endpoint on a registered instance by looking up its homePageUrl entry in Eureka, then appending /hystrix.stream to it. This means that if spring-boot-actuator is running on its own port (which is the default), the call to /hystrix.stream will fail. To make turbine find the Hystrix stream at the correct port, you need to add management.port to the instances' metadata:
> 
> 	eureka:
> 	  instance:
> 	    metadata-map:
> 	      management.port: ${management.port:8081}


The configuration key turbine.appConfig is a list of eureka serviceIds that turbine will use to lookup instances. The turbine stream is then used in the Hystrix dashboard using a url that looks like: http://my.turbine.sever:8080/turbine.stream?cluster=<CLUSTERNAME>; (the cluster parameter can be omitted if the name is "default"). The cluster parameter must match an entry in turbine.aggregator.clusterConfig. Values returned from eureka are uppercase, thus we expect this example to work if there is an app registered with Eureka called "customers":

turbine.appConfig配置是一个eureka服务ID列表，turbine将使用这个配置查询实例。turbine stream在hystrix dashboard中使用如下的url配置： http://my.turbine.server:8080/turbine.stream?cluster=,如果集群的名称是default，集群参数可以忽略）。这个cluster参数必须和turbine.aggregator.clusterConfig匹配。从eureka返回的值都是大写的，因此我们希望下面的例子可以工作，如果一个app使用eureka注册，并且被叫做”customers

	turbine:
	  aggregator:
	    clusterConfig: CUSTOMERS
	  appConfig: customers

The clusterName can be customized by a SPEL expression in turbine.clusterNameExpression with root an instance of InstanceInfo. The default value is appName, which means that the Eureka serviceId ends up as the cluster key (i.e. the InstanceInfo for customers has an appName of "CUSTOMERS"). A different example would be turbine.clusterNameExpression=aSGName, which would get the cluster name from the AWS ASG name. Another example:

clusterName可以使用SPEL表达式定义，在turbine.clusterNameExpression。 默认值是appName,意思是eureka服务ID最终将作为集群的key，例如customers的 InstanceInfo有一个CUSTOMERS的appName。另外一个例子是turbine.clusterNameExpression=aSGName，将从AWS ASG name获取集群名称。另一个例子:

	turbine:
	  aggregator:
	    clusterConfig: SYSTEM,USER
	  appConfig: customers,stores,ui,admin
	  clusterNameExpression: metadata['cluster']

In this case, the cluster name from 4 services is pulled from their metadata map, and is expected to have values that include "SYSTEM" and "USER".

在这种情况下,集群名称从4个服务从其元数据映射,期望包含“SYSTEM”和“USER”。

To use the "default" cluster for all apps you need a string literal expression (with single quotes, and escaped with double quotes if it is in YAML as well):

	turbine:
	  appConfig: customers,stores
	  clusterNameExpression: "'default'"

Spring Cloud provides a spring-cloud-starter-turbine that has all the dependencies you need to get a Turbine server running. Just create a Spring Boot application and annotate it with @EnableTurbine.

spring cloud提供一个spring-cloud-starter-turbine,它拥有你需要运行一个turbine服务器的所有依赖项。您仅需要使用@EnableTurbine创建一个spring boot应用。

> NOTE
> by default Spring Cloud allows Turbine to use the host and port to allow multiple processes per host, per cluster. If you want the native Netflix behaviour built into Turbine that does not allow multiple processes per host, per cluster (the key to the instance id is the hostname), then set the property turbine.combineHostPort=false.
> 注意：默认情况下Spring Cloud 允许 Turbine 在集群的每个主机下使用主机名和端口运行多个进程。如果你想在集群中的每个主机使用本机原生Netfix行为且不允许多个进程创建运行Turbine。(实例id的key为主机名）然后设置属性turbine.combineHostPort=false
> 

### Turbine Stream
In some environments (e.g. in a PaaS setting), the classic Turbine model of pulling metrics from all the distributed Hystrix commands doesn’t work. In that case you might want to have your Hystrix commands push metrics to Turbine, and Spring Cloud enables that with messaging. All you need to do on the client is add a dependency to spring-cloud-netflix-hystrix-stream and the spring-cloud-starter-stream-* of your choice (see Spring Cloud Stream documentation for details on the brokers, and how to configure the client credentials, but it should work out of the box for a local broker).

在一些环境(Pass), 在所有分布式下典型的Turbine 模型的Hystrix 命令都不工作，在这种情况下，你可能想要 Hystrix 命令 推送到 Tuibine， 和Spring Cloud进行消息传递，那么你需要做的是在客户端添加一个依赖spring-cloud-netflix-hystrix-stream和你所选择的 spring-cloud-starter-stream-*的依赖(相关信息请查看 Spring Cloud Stream 方档，以及如何配置客户端凭证，和工作时的要本地代理)


On the server side Just create a Spring Boot application and annotate it with @EnableTurbineStream and by default it will come up on port 8989 (point your Hystrix dashboard to that port, any path). You can customize the port using either server.port or turbine.stream.port. If you have spring-boot-starter-web and spring-boot-starter-actuator on the classpath as well, then you can open up the Actuator endpoints on a separate port (with Tomcat by default) by providing a management.port which is different.

创建一个带有注解 @EnableTurbineStream 的Spring boot 应用服务器，端口默认 8989 (Hystrix 仪表盘的URL都使用此端口), 如果你想自定义端口，可以配置 server.port 或 turbine.stream.port 任一个，如果你使用了 spring-boot-starter-web 和 spring-boot-starter-actuator ，那么你可以提供(使用Tomcat默认情况下) management.port 不同的端口，并打开这个单独的执行器端口。

You can then point the Hystrix Dashboard to the Turbine Stream Server instead of individual Hystrix streams. If Turbine Stream is running on port 8989 on myhost, then put http://myhost:8989 in the stream input field in the Hystrix Dashboard. Circuits will be prefixed by their respective serviceId, followed by a dot, then the circuit name.

你可以把Dashboard指向Turbine Stream Server来代替个别Hystrix streams。如果Tubine Stream 使用你本机的8989端口运行，然后把 http://myhost:8989在流输入字段Hystrix仪表板 Circuits 将由各自的 serverId关缀，紧随其后的是一个点，然后是circuit 名称。


Spring Cloud provides a spring-cloud-starter-turbine-stream that has all the dependencies you need to get a Turbine Stream server running - just add the Stream binder of your choice, e.g. spring-cloud-starter-stream-rabbit. You need Java 8 to run the app because it is Netty-based.

Spring Cloud提供了一个 spring-cloud-starter-turbine-stream，包括了运行 Turibine Stream Server运行所需要的所有依赖,如spring-cloud-starter-stream-rabbit。因为是基于Netty的，所以你需要Java 8 运行此应用
