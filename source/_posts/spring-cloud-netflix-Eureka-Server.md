---
title: Spring Cloud Netflix--Eureka Server
date: 2017-03-10 11:18:20
description: 
categories:
- Spring Cloud Netflix
tags:
- Service Discovery
- Eureka Server
toc: true
author: John Tang
comments:
original:
permalink: 
---

## Service Discovery: Eureka Server 服务发现

### How to Include Eureka Server
To include Eureka Server in your project use the starter with group org.springframework.cloud and artifact id spring-cloud-starter-eureka-server. See the Spring Cloud Project page for details on setting up your build system with the current Spring Cloud Release Train.

通过在项目pom.xml中引入group为org.springframework.cloud 和 artifact id为spring-cloud-starter-eureka-server可以快速引用Eureka Server。当您需要使用当前Spring Cloud 发布版本配置您的项目时，请参考Spring Cloud 项目页面以获取更详细信息。

### How to Run a Eureka Server 如何运行Eureka Server

Example eureka server;

	@SpringBootApplication
	@EnableEurekaServer
	public class Application {
	
	    public static void main(String[] args) {
	        new SpringApplicationBuilder(Application.class).web(true).run(args);
	    }
	
	}

The server has a home page with a UI, and HTTP API endpoints per the normal Eureka functionality under /eureka/*.

服务拥有一个基于HTTP API 接口的UI页面，接口地址基于/eureka/* 。
<!-- more -->
Eureka background reading: see flux capacitor and google group discussion.



> TIP
> Due to Gradle’s dependency resolution rules and the lack of a parent bom feature, simply depending on spring-cloud-starter-eureka-server can cause failures on application startup. To remedy this the Spring Boot Gradle plugin must be added and the Spring cloud starter parent bom must be imported like so:
> 
> 由于Gradle的依赖解析规则缺乏父依赖，单纯的依靠spring-cloud-starter-eureka-server会导致启动失败。为了填补这一缺陷，必须添加Spring Boot Gradle插件和导入Spring cloud starter的父依赖,如:
> 
> build.gradle
> 
> 	buildscript {
> 	  dependencies {
> 	    classpath("org.springframework.boot:spring-boot-gradle-plugin:1.3.5.RELEASE")
> 	  }
> 	}
> 
> apply plugin: "spring-boot"
> 
> 	dependencyManagement {
> 	  imports {
> 	    mavenBom "org.springframework.cloud:spring-cloud-dependencies:Brixton.RELEASE"
> 	  }
> 	}


### High Availability, Zones and Regions 高可用
The Eureka server does not have a backend store, but the service instances in the registry all have to send heartbeats to keep their registrations up to date (so this can be done in memory). Clients also have an in-memory cache of eureka registrations (so they don’t have to go to the registry for every single request to a service).

虽然Eureka服务端没有后台存储，但是在注册表里的服务实例全都得发送心跳去保持注册更新（在内存里操作）。客户端也有一份内存缓存着eureka的注册信息（因此，他们不必表为每单一的请求注册到一个服务）。

By default every Eureka server is also a Eureka client and requires (at least one) service URL to locate a peer. If you don’t provide it the service will run and work, but it will shower your logs with a lot of noise about not being able to register with the peer.

默认情况下每个Eureka服务端也是一个Eureka客户端并且通过请求服务的URL去定位一个节点。如果你不提供的话，服务虽然还会运行和工作，但是它不会向你打印一堆关于没有注册的节点日志。

See also below for details of Ribbon support on the client side for Zones and Regions.

请参阅 以下关于 Ribbon 对client 和zone 的处理细节。

### Standalone Mode 单机模式
The combination of the two caches (client and server) and the heartbeats make a standalone Eureka server fairly resilient to failure, as long as there is some sort of monitor or elastic runtime keeping it alive (e.g. Cloud Foundry). In standalone mode, you might prefer to switch off the client side behaviour, so it doesn’t keep trying and failing to reach its peers. Example:

这两个缓存的组合（客户端和服务器）和心跳 使一个单机模式的Eureka服务能快速恢复，只要有相应的监控或者弹性时间来保持它活着。（如 Cloud Foundry）。 在单机模式，您可能更倾向于关闭客户端的行为，所以它不能保持失败重试和重新找回它的那些节点。如：


application.yml (Standalone Eureka Server)

	server:
	  port: 8761
	
	eureka:
	  instance:
	    hostname: localhost
	  client:
	    registerWithEureka: false
	    fetchRegistry: false
	    serviceUrl:
	      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

Notice that the serviceUrl is pointing to the same host as the local instance.

请注意，serviceUrl是指向同一个主机的本地实例。

### Peer Awareness 节点感知
Eureka can be made even more resilient and available by running multiple instances and asking them to register with each other. In fact, this is the default behaviour, so all you need to do to make it work is add a valid serviceUrl to a peer, e.g.

Eureka甚至可以更有弹性和可用的运行多个实例,并让他们互相注册。事实上，这也是默认的行为，因此你需要做的是，只要添加一个有效的节点serviceUrl来让它工作，例如：

application.yml (Two Peer Aware Eureka Servers)

	---
	spring:
	  profiles: peer1
	eureka:
	  instance:
	    hostname: peer1
	  client:
	    serviceUrl:
	      defaultZone: http://peer2/eureka/
	
	---
	spring:
	  profiles: peer2
	eureka:
	  instance:
	    hostname: peer2
	  client:
	    serviceUrl:
	      defaultZone: http://peer1/eureka/

In this example we have a YAML file that can be used to run the same server on 2 hosts (peer1 and peer2), by running it in different Spring profiles. You could use this configuration to test the peer awareness on a single host (there’s not much value in doing that in production) by manipulating /etc/hosts to resolve the host names. In fact, the eureka.instance.hostname is not needed if you are running on a machine that knows its own hostname (it is looked up using java.net.InetAddress by default).

在这个例子，我们有一个YAML文件，，能够被使用于在不同的的Spring profiles，运行相同的服务的两个主机（peer1 和 peer2）。你还可以通过管理/etc/hosts来解析主机名称的方式，使用这个配置在单个主机上去测试节点感知（在生产中这样做没有多少价值）。实际上，如果你在一台已知自身hostname的机器上运行的话，eureka.instance.hostname是不需要的（默认查找使用的java.net.InetAddress）。

You can add multiple peers to a system, and as long as they are all connected to each other by at least one edge, they will synchronize the registrations amongst themselves. If the peers are physically separated (inside a data centre or between multiple data centres) then the system can in principle survive split-brain type failures.

你可以添加更多的节点进一个系统，并且只要他们可以通过至少一端来互相连接，他们将同步自身的互相注册信息。如果节点被分离（在一个数据中心或者多个数据中心之间），那么，系统原则上 survive split-brain type failures。

### Prefer IP Address  使用IP地址
In some cases, it is preferable for Eureka to advertise the IP Adresses of services rather than the hostname. Set eureka.instance.preferIpAddress to true and when the application registers with eureka, it will use its IP Address rather than its hostname.

有些时候你可能更倾向于直接使用IP地址定义服务而不是使用主机名。把eureka.instance.preferIpAddress参数设为true时，客户端在注册时就会使用自己的ip地址而不是主机名。
