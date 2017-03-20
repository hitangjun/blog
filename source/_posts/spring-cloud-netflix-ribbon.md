---
title: Spring Cloud Netflix--Ribbon
date: 2017-03-10 15:55:18
description: 
categories:
- Spring Cloud Netflix
tags:
- Ribbon
toc: true
author: John Tang
comments:
original:
permalink: 
---

## Client Side Load Balancer: Ribbon 客户端负载均衡：Ribbon
Ribbon is a client side load balancer which gives you a lot of control over the behaviour of HTTP and TCP clients. Feign already uses Ribbon, so if you are using @FeignClient then this section also applies.

Ribbon是一个客户端的负载均衡器，可以提供很多HTTP和TCP的控制行为。Feign已经使用了Ribbon，所以如果你使用了@FeignClient，Riboon也同样被应用了。
<!-- more -->
A central concept in Ribbon is that of the named client. Each load balancer is part of an ensemble of components that work together to contact a remote server on demand, and the ensemble has a name that you give it as an application developer (e.g. using the @FeignClient annotation). 

Ribbon核心的概念是named client。 每个负载均衡器都是共同体的一部分，可以一起运行去连接远程服务器，你会给你的应用设置一个名字（比如使用@FeignClient注解）。

Spring Cloud creates a new ensemble as an ApplicationContext on demand for each named client using RibbonClientConfiguration. This contains (amongst other things) an ILoadBalancer, a RestClient, and a ServerListFilter.

Spring Cloud使用RibbonClientConfiguration为每一个命名的客户端建立一个新系列为满足ApplicationContext的需求。 这包括一个ILoadBalancer， 一个RestClient和一个ServerListFilter。



### How to Include Ribbon 如何引入Ribbon
To include Ribbon in your project use the starter with group org.springframework.cloud and artifact id spring-cloud-starter-ribbon. See the Spring Cloud Project page for details on setting up your build system with the current Spring Cloud Release Train.

为了引入Ribbon到你的工程，可以使用org.springframework.cloud组的starter.artifact id是spring-cloud-starter-ribbon。可以参照Spring Cloud Project的细节来设置你的构建系统。

### Customizing the Ribbon Client 定制化Ribbon Client
You can configure some bits of a Ribbon client using external properties in <client>.ribbon.*, which is no different than using the Netflix APIs natively, except that you can use Spring Boot configuration files. The native options can be inspected as static fields in CommonClientConfigKey (part of ribbon-core).

你可以使用外部的属性<client>.ribbon.*来配置一些Ribbon Client，除了能够使用Sprint Boot的配置文件以外，它和使用原生的Netflix APIS没什么不同。原生选项可以在CommonClientConfigKey中被检查到。

Spring Cloud also lets you take full control of the client by declaring additional configuration (on top of the RibbonClientConfiguration) using @RibbonClient. Example:

通过使用@RibbonClient声明额外的配置（在RibbonClientConfiguration上面），Spring Cloud也让你全面的控制Client。 例如：

	@Configuration
	@RibbonClient(name = "foo", configuration = FooConfiguration.class)
	public class TestConfiguration {
	}

In this case the client is composed from the components already in RibbonClientConfiguration together with any in FooConfiguration (where the latter generally will override the former).

这个例子中，Client除了包括了已经存在于RibbonClientConfiguration的组件，也包括了自定义的FooConfiguration（一般后者会重载前者）。

> WARNING
> The FooConfiguration has to be @Configuration but take care that it is not in a @ComponentScan for the main application context, otherwise it will be shared by all the @RibbonClients. If you use @ComponentScan (or @SpringBootApplication) you need to take steps to avoid it being included (for instance put it in a separate, non-overlapping package, or specify the packages to scan explicitly in the @ComponentScan).
>
>警告：FooConfiguration 不能被@ComponentScan 在main application context。这样的话，它将被所有@RibbonClients共享。如果你使用 @ComponentScan (or @SpringBootApplication) ，你需要避免它被包括其中。(例如：放它到一个独立的，无重叠的包里，或者指明不被@ComponentScan扫描)。

Spring Cloud Netflix provides the following beans by default for ribbon (BeanType beanName: ClassName):

Spring Cloud Netflix为ribbon提供了如下的Beans(BeanType beanName: ClassName):

	IClientConfig ribbonClientConfig: DefaultClientConfigImpl
	
	IRule ribbonRule: ZoneAvoidanceRule
	
	IPing ribbonPing: NoOpPing
	
	ServerList<Server> ribbonServerList: ConfigurationBasedServerList
	
	ServerListFilter<Server> ribbonServerListFilter: ZonePreferenceServerListFilter
	
	ILoadBalancer ribbonLoadBalancer: ZoneAwareLoadBalancer

Creating a bean of one of those type and placing it in a @RibbonClient configuration (such as FooConfiguration above) allows you to override each one of the beans described. Example:

建立一个如上类型的bean，放置到@RibbonClient配置（如上FooConfiguration），允许你重载其中的任意一个。例如：

	@Configuration
	public class FooConfiguration {
	    @Bean
	    public IPing ribbonPing(IClientConfig config) {
	        return new PingUrl();
	    }
	}

This replaces the NoOpPing with PingUrl.

如上将用PingUrl代替NoOpPing。

### Customizing the Ribbon Client using properties 使用properties定制化Ribbon Client
Starting with version 1.2.0, Spring Cloud Netflix now supports customizing Ribbon clients using properties to be compatible with the Ribbon documentation.
This allows you to change behavior at start up time in different environments.
The supported properties are listed below and should be prefixed by <clientName>.ribbon.:

从1.2.0开始，Spring Cloud Netflix支持使用properties来定制化Ribbon clients。 
这就允许你在不同的环境，在启动时改变行为。
支持的属性如下（加上.ribbon.的前缀）:

	NFLoadBalancerClassName: should implement ILoadBalancer
	
	NFLoadBalancerRuleClassName: should implement IRule
	
	NFLoadBalancerPingClassName: should implement IPing
	
	NIWSServerListClassName: should implement ServerList
	
	NIWSServerListFilterClassName should implement ServerListFilter

> NOTE
> Classes defined in these properties have precedence over beans defined using @RibbonClient(configuration=MyRibbonConfig.class) and the defaults provided by Spring Cloud Netflix.
>
>在这些属性中定义的Classes优先于在@RibbonClient(configuration=MyRibbonConfig.class)定义的beans和由Spring Cloud Netflix提供的缺省值。

To set the IRule for a service name users you could set the following:

例如，为一个叫user的服务设置IRule，可以进行如下设置：

application.yml

	users:
	  ribbon:
	    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule

See the Ribbon documentation for implementations provided by Ribbon.

### Using Ribbon with Eureka
When Eureka is used in conjunction with Ribbon (i.e., both are on the classpath) the ribbonServerList is overridden with an extension of DiscoveryEnabledNIWSServerList which populates the list of servers from Eureka. It also replaces the IPing interface with NIWSDiscoveryPing which delegates to Eureka to determine if a server is up. The ServerList that is installed by default is a DomainExtractingServerList and the purpose of this is to make physical metadata available to the load balancer without using AWS AMI metadata (which is what Netflix relies on). By default the server list will be constructed with "zone" information as provided in the instance metadata (so on the remote clients set eureka.instance.metadataMap.zone), and if that is missing it can use the domain name from the server hostname as a proxy for zone (if the flag approximateZoneFromHostname is set). Once the zone information is available it can be used in a ServerListFilter. By default it will be used to locate a server in the same zone as the client because the default is a ZonePreferenceServerListFilter. The zone of the client is determined the same way as the remote instances by default, i.e. via eureka.instance.metadataMap.zone.

当Eureka被和Ribbon一起联合使用时， ribbonServerList被一个叫做DiscoveryEnabledNIWSServerList的扩展重载。这个扩展汇聚了来自于Eureka的服务列表。 IPing的接口也被NIWSDiscoveryPing代替，用于委托Eureka来测定服务是否UP。 缺省的ServerListThe是一个DomainExtractingServerList，目的是使得物理上的元数据可以用于负载均衡器，而不必要使用AWA AMI的元数据。 缺省情况下，服务List将根据“zone”信息被创建。（远程客户端设置eureka.instance.metadataMap.zone）。如果这个没有设置，它可以使用d来自于服务hostname的domain name作为zone的代理（取决于approximateZoneFromHostname是否被设置）。一旦那个zone的信息可获得，他可以在ServerListFilter被使用。缺省情况下，由于使用ZonePreferenceServerListFilter，它被用于定位一个和Client相同zone的server。 默认情况下，client的zone是和远程实例一样的，通过eureka.instance.metadataMap.zone来设置。



> NOTE
> The orthodox "archaius" way to set the client zone is via a configuration property called "@zone", and Spring Cloud will use that in preference to all other settings if it is available (note that the key will have to be quoted in YAML configuration).
> .
>
> 注意：正统的"archaius"方式是通过配置属性"@zone"来设置Clinet zone。如果它是有效的，Spring Cloud使用这个优先于其他设置。（注意，这个key必须在YAML configuration被引)。
 
.
> NOTE
> If there is no other source of zone data then a guess is made based on the client configuration (as opposed to the instance configuration). We take eureka.client.availabilityZones, which is a map from region name to a list of zones, and pull out the first zone for the instance’s own region (i.e. the eureka.client.region, which defaults to "us-east-1" for comatibility with native Netflix).
> .
>
> 如果zone数据没有其他来源，基于Client的配置将会有一个猜测。(as opposed to the instance configuration).我们称为eureka.client.availabilityZones。它是一个从region名到zones列表的map，并且抽出第一个zone 作为实例自己的region（例如the eureka.client.region）


#### Example: How to Use Ribbon Without Eureka
Eureka is a convenient way to abstract the discovery of remote servers so you don’t have to hard code their URLs in clients, but if you prefer not to use it, Ribbon and Feign are still quite amenable. Suppose you have declared a @RibbonClient for "stores", and Eureka is not in use (and not even on the classpath). The Ribbon client defaults to a configured server list, and you can supply the configuration like this

Example: 没有Eureka，如何使用Ribbon
Eureka是一个实用的方式，它抽象了远程服务的发现机制，可以使你不用在Clinet中硬编码URLS。但如果你不打算使用它，Ribbon和Feign仍然是相当可用的。假设你已经声明了一个@RibbonClient为一个“Stores"的服务。Eureka没有被使用。Ribbon Client会缺省到一个已配置的Server列表。你可以提供配置像：

application.yml
stores:
  ribbon:
    listOfServers: example.com,google.com

#### Example: Disable Eureka use in Ribbon
Setting the property ribbon.eureka.enabled = false will explicitly disable the use of Eureka in Ribbon.

设置ribbon.eureka.enabled = false，可以在Ribbon中停止对Eureka的使用

application.yml

	ribbon:
	  eureka:
	   enabled: false

### Using the Ribbon API Directly 直接使用Ribbon API
You can also use the LoadBalancerClient directly. Example:

	public class MyClass {
	    @Autowired
	    private LoadBalancerClient loadBalancer;
	
	    public void doStuff() {
	        ServiceInstance instance = loadBalancer.choose("stores");
	        URI storesUri = URI.create(String.format("http://%s:%s", instance.getHost(), instance.getPort()));
	        // ... do something with the URI
	    }
	}
