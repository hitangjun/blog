---
title: Spring Cloud Netflix--Feign
date: 2017-03-10 16:55:18
description: 
categories:
- Spring Cloud Netflix
tags:
- Feign
toc: true
author: John Tang
comments:
original:
permalink: 
---

## Declarative REST Client: Feign 声明式REST客户端：Feign
Feign is a declarative web service client. It makes writing web service clients easier. To use Feign create an interface and annotate it. It has pluggable annotation support including Feign annotations and JAX-RS annotations. Feign also supports pluggable encoders and decoders. Spring Cloud adds support for Spring MVC annotations and for using the same HttpMessageConverters used by default in Spring Web. Spring Cloud integrates Ribbon and Eureka to provide a load balanced http client when using Feign.

Feign是一个声明式Web Service客户端。使用Feign能让编写Web Service客户端更加简单, 它的使用方法是定义一个接口，然后在上面添加注解，同时也支持JAX-RS标准的注解。Feign也支持可拔插式的编码器和解码器。Spring Cloud对Feign进行了封装，使其支持了Spring MVC标准注解和HttpMessageConverters。Feign可以与Eureka和Ribbon组合使用以支持负载均衡。

<!-- more -->

### How to Include Feign
To include Feign in your project use the starter with group org.springframework.cloud and artifact id spring-cloud-starter-feign. See the Spring Cloud Project page for details on setting up your build system with the current Spring Cloud Release Train.

Example spring boot app

	@Configuration
	@ComponentScan
	@EnableAutoConfiguration
	@EnableEurekaClient
	@EnableFeignClients
	public class Application {
	
	    public static void main(String[] args) {
	        SpringApplication.run(Application.class, args);
	    }
	
	}

StoreClient.java

	@FeignClient("stores")
	public interface StoreClient {
	    @RequestMapping(method = RequestMethod.GET, value = "/stores")
	    List<Store> getStores();
	
	    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
	    Store update(@PathVariable("storeId") Long storeId, Store store);
	}

In the @FeignClient annotation the String value ("stores" above) is an arbitrary client name, which is used to create a Ribbon load balancer (see below for details of Ribbon support). You can also specify a URL using the url attribute (absolute value or just a hostname). The name of the bean in the application context is the fully qualified name of the interface. To specify your own alias value you can use the qualifier value of the @FeignClient annotation.

@FeignClient注解中的stores属性可以是一个任意字符串(译者注：如果与Eureka组合使用，则stores应为Eureka中的服务名)，Feign用它来创建一个Ribbon负载均衡器。你也可以通过url属性来指定一个地址(可以是完整的URL，也可以是一个主机名)。标注了@FeignClient注解的接口在ApplicationContext中的Bean实例名是这个接口的全限定名，同时这个Bean还有一个别名，为Bean名 + FeignClient。在本例中，可以使用@Qualifier("storesFeignClient")来注入该组件。


The Ribbon client above will want to discover the physical addresses for the "stores" service. If your application is a Eureka client then it will resolve the service in the Eureka service registry. If you don’t want to use Eureka, you can simply configure a list of servers in your external configuration (see above for example).

如果classpath中有Ribbon, 上面的例子中Ribbon Client会想办法查找stores服务的IP地址。如果Eureka也在classpath中，那么Ribbon会从Eureka的注册信息中查找。如果你不想用Eureka,你也可以在配置文件中直接指定一组服务器地址。

### Overriding Feign Defaults 覆盖Feign的默认配置
A central concept in Spring Cloud’s Feign support is that of the named client. Each feign client is part of an ensemble of components that work together to contact a remote server on demand, and the ensemble has a name that you give it as an application developer using the @FeignClient annotation. Spring Cloud creates a new ensemble as an ApplicationContext on demand for each named client using FeignClientsConfiguration. This contains (amongst other things) an feign.Decoder, a feign.Encoder, and a feign.Contract.

Spring Cloud对Feign的封装中一个核心的概念就是客户端要有一个名字。每一个客户端随时可以向远程服务发起请求，并且每个服务都可以像使用@FeignClient注解一样指定一个名字。Spring Cloud会将所有的@FeignClient组合在一起创建一个新的ApplicationContext, 并使用FeignClientsConfiguration 对 Clients 进行配置。配置中包括编码器、解码器和 feign.Contract 。

Spring Cloud lets you take full control of the feign client by declaring additional configuration (on top of the FeignClientsConfiguration) using @FeignClient. Example:

Spring Cloud允许你通过configuration属性完全控制Feign的配置信息，这些配置比FeignClientsConfiguration优先级要高：

	@FeignClient(name = "stores", configuration = FooConfiguration.class)
	public interface StoreClient {
	    //..
	}

In this case the client is composed from the components already in FeignClientsConfiguration together with any in FooConfiguration (where the latter will override the former).

在这个例子中，FooConfiguration中的配置信息会覆盖掉FeignClientsConfiguration中对应的配置。

> NOTE
> FooConfiguration does not need to be annotated with @Configuration. However, if it is, then take care to exclude it from any @ComponentScan that would otherwise include this configuration as it will become the default source for feign.Decoder, feign.Encoder, feign.Contract, etc., when specified. This can be avoided by putting it in a separate, non-overlapping package from any @ComponentScan or @SpringBootApplication, or it can be explicitly excluded in @ComponentScan.
> 警告: FooConfiguration必须是@Configuration，但是注意不能在@CompinentScan中，否则将被用于每个@FeignClient。如果你使用@ComponentScan（或@ SpringBootApplication），你需要采取一些措施来避免它被列入（比如把它放在一个单独的，非重叠的包，或者指定包在@ComponentScan明确扫描）。
> 
.
> NOTE
> The serviceId attribute is now deprecated in favor of the name attribute.
> 注意：serviceId属性已经被弃用了，取而代之的是name属性。
.
> WARNING
> Previously, using the url attribute, did not require the name attribute. Using name is now required.
> 注意：
> 在先前的版本中在指定了url属性时name是可选属性，现在无论什么时候name都是必填属性。

Placeholders are supported in the name and url attributes.

name 和 url 属性都支持占位符。

	@FeignClient(name = "${feign.name}", url = "${feign.url}")
	public interface StoreClient {
	    //..
	}

Spring Cloud Netflix provides the following beans by default for feign (BeanType beanName: ClassName):

Spring Cloud Netflix为Feign提供了以下默认的配置Bean：(下面最左侧是Bean的类型，中间是Bean的name, 右侧是类名)

	Decoder feignDecoder: ResponseEntityDecoder (which wraps a SpringDecoder)
	
	Encoder feignEncoder: SpringEncoder
	
	Logger feignLogger: Slf4jLogger
	
	Contract feignContract: SpringMvcContract
	
	Feign.Builder feignBuilder: HystrixFeign.Builder
	
	Client feignClient: if Ribbon is enabled it is a LoadBalancerFeignClient, otherwise the default feign client is used.

The OkHttpClient and ApacheHttpClient feign clients can be used by setting feign.okhttp.enabled or feign.httpclient.enabled to true, respectively, and having them on the classpath.

OkHttpClient和ApacheHttpClient feign clients可以通过分别设置fiegn.okhttp.enable 或者 feign.httpclient.enable为true，并且添加到classpath。

Spring Cloud Netflix does not provide the following beans by default for feign, but still looks up beans of these types from the application context to create the feign client:

Spring cloud netflix默认没有提供一下bean，但是仍然可以从上下文中查找这些bean并创建feign client： 

	Logger.Level
	
	Retryer
	
	ErrorDecoder
	
	Request.Options
	
	Collection<RequestInterceptor>
	
	SetterFactory

Creating a bean of one of those type and placing it in a @FeignClient configuration (such as FooConfiguration above) allows you to override each one of the beans described. Example:

创建这些类型的一个bean可以放在@FeignClient配置中(如上FooConfiguration),允许你覆盖所描述的每一个bean. 例子:

	@Configuration
	public class FooConfiguration {
	    @Bean
	    public Contract feignContract() {
	        return new feign.Contract.Default();
	    }
	
	    @Bean
	    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
	        return new BasicAuthRequestInterceptor("user", "password");
	    }
	}

This replaces the SpringMvcContract with feign.Contract.Default and adds a RequestInterceptor to the collection of RequestInterceptor.

可以替换SpringMvcContract 和 feign.Contract.Default, 并增加一个 RequestInterceptor 到 RequestInterceptor 中去.

Default configurations can be specified in the @EnableFeignClients attribute defaultConfiguration in a similar manner as described above. The difference is that this configuration will apply to all feign clients.

可以通过@EnableFeignClients的属性defaultConfiguration以同样的方式被指定。不同之处是配置会加载到所有的feign clients。

> NOTE
> If you need to use ThreadLocal bound variables in your RequestInterceptor`s you will need to either set the thread isolation strategy for Hystrix to `SEMAPHORE or disable Hystrix in Feign.

application.yml

	# To disable Hystrix in Feign
	feign:
	  hystrix:
	    enabled: false
	
	# To set thread isolation to SEMAPHORE
	hystrix:
	  command:
	    default:
	      execution:
	        isolation:
	          strategy: SEMAPHORE

### Creating Feign Clients Manually 自主创建Feign Clients
In some cases it might be necessary to customize your Feign Clients in a way that is not possible using the methods above. In this case you can create Clients using the Feign Builder API. Below is an example which creates two Feign Clients with the same interface but configures each one with a separate request interceptor.

在一些情况下可能需要自定义Feign clients，但是却不能用以上的方法。所以你可以使用Feign Builder API创建clients。下面是一个例子，创建了两个相同接口的client但是分别配置了不同的拦截器。

	@Import(FeignClientsConfiguration.class)
	class FooController {
	
		private FooClient fooClient;
	
		private FooClient adminClient;
	
	    @Autowired
		public FooController(
				Decoder decoder, Encoder encoder, Client client) {
			this.fooClient = Feign.builder().client(client)
					.encoder(encoder)
					.decoder(decoder)
					.requestInterceptor(new BasicAuthRequestInterceptor("user", "user"))
					.target(FooClient.class, "http://PROD-SVC");
			this.adminClient = Feign.builder().client(client)
					.encoder(encoder)
					.decoder(decoder)
					.requestInterceptor(new BasicAuthRequestInterceptor("admin", "admin"))
					.target(FooClient.class, "http://PROD-SVC");
	    }
	}

> NOTE
> In the above example FeignClientsConfiguration.class is the default configuration provided by Spring Cloud Netflix.
> NOTE
> PROD-SVC is the name of the service the Clients will be making requests to.
> 
> 注意：在这个例子中，FeignClientsConfiguration.class是Spring Cloud Netflix默认提供的配置。
> 
> PROD-SVC是我们提供的服务名称，会接收相应的客户端的请求。

### Feign Hystrix Support
If Hystrix is on the classpath and feign.hystrix.enabled=true, Feign will wrap all methods with a circuit breaker. Returning a com.netflix.hystrix.HystrixCommand is also available. This lets you use reactive patterns (with a call to .toObservable() or .observe() or asynchronous use (with a call to .queue()).

如果Hystrix在classpath中，默认Feign用熔断器包装所有方法。返回一个 com.netflix.hystrix.HystrixCommand 也是可用的。这允许你以被动模式使用（使用.toObservable()或者.observer()）或者 异步调用（.queue()）。


To disable Hystrix support on a per-client basis create a vanilla Feign.Builder with the "prototype" scope, e.g.:


要禁用Feign 的 Hystrix支持，设置feign.hystrix.enabled=false.

要在每个客户端上禁用 Hystrix 支持，创建一个 Feign.Builder 并将scope 设置为”prototype”,例如:

	@Configuration
	public class FooConfiguration {
	    @Bean
		@Scope("prototype")
		public Feign.Builder feignBuilder() {
			return Feign.builder();
		}
	}

> WARNING
> 
> Prior to the Spring Cloud Dalston release, if Hystrix was on the classpath Feign would have wrapped all methods in a circuit breaker by default. This default behavior was changed in Spring Cloud Dalston in favor for an opt-in approach.

### Feign Hystrix Fallbacks

Hystrix supports the notion of a fallback: a default code path that is executed when they circuit is open or there is an error. To enable fallbacks for a given @FeignClient set the fallback attribute to the class name that implements the fallback.

Hystrix支持回退的概念：一段默认的代码将会被执行当断路器打开或者发生错误。要启用回退要给@FeignClient设置fallback属性来实现回退.

	@FeignClient(name = "hello", fallback = HystrixClientFallback.class)
	protected interface HystrixClient {
	    @RequestMapping(method = RequestMethod.GET, value = "/hello")
	    Hello iFailSometimes();
	}
	
	static class HystrixClientFallback implements HystrixClient {
	    @Override
	    public Hello iFailSometimes() {
	        return new Hello("fallback");
	    }
	}

If one needs access to the cause that made the fallback trigger, one can use the fallbackFactory attribute inside @FeignClient.

如果一个请求需要触发回退，可以使用fallbackFactory属性替换@FeignClient。

	@FeignClient(name = "hello", fallbackFactory = HystrixClientFallbackFactory.class)
	protected interface HystrixClient {
		@RequestMapping(method = RequestMethod.GET, value = "/hello")
		Hello iFailSometimes();
	}
	
	@Component
	static class HystrixClientFallbackFactory implements FallbackFactory<HystrixClient> {
		@Override
		public HystrixClient create(Throwable cause) {
			return new HystrixClientWithFallBackFactory() {
				@Override
				public Hello iFailSometimes() {
					return new Hello("fallback; reason was: " + cause.getMessage());
				}
			};
		}
	}

> WARNING
> There is a limitation with the implementation of fallbacks in Feign and how Hystrix fallbacks work. Fallbacks are currently not supported for methods that return com.netflix.hystrix.HystrixCommand and rx.Observable.
> Feign and @Primary
> When using Feign with Hystrix fallbacks, there are multiple beans in the ApplicationContext of the same type. This will cause @Autowired to work because there isn’t exactly one bean, or one marked as primary. To work around this, Spring Cloud Netflix marks all Feign instances as @Primary, so Spring Framework will know which bean to inject. In some cases, this may not be desirable. To turn off this behavior set the primary attribute of @FeignClient to false.

	@FeignClient(name = "hello", primary = false)
	public interface HelloClient {
		// methods here
	}

### Feign Inheritance Support
Feign supports boilerplate apis via single-inheritance interfaces. This allows grouping common operations into convenient base interfaces.

UserService.java
	public interface UserService {
	
	    @RequestMapping(method = RequestMethod.GET, value ="/users/{id}")
	    User getUser(@PathVariable("id") long id);
	}

UserResource.java

	@RestController
	public class UserResource implements UserService {
	
	}
	UserClient.java
	package project.user;
	
	@FeignClient("users")
	public interface UserClient extends UserService {
	
	}

> NOTE
> It is generally not advisable to share an interface between a server and a client. It introduces tight coupling, and also actually doesn’t work with Spring MVC in its current form (method parameter mapping is not inherited).
> 
> 注意：通常在一个server和一个client之间共享一个接口是不可取的。它引入了紧耦合，实际上它也不会spring mvc中起作用（方法参数映射不会被继承）。

### Feign request/response compression

You may consider enabling the request or response GZIP compression for your Feign requests. You can do this by enabling one of the properties:

你可能考虑对你的Feign请求启用GZIP压缩。你可以通过设置如下启用：


	feign.compression.request.enabled=true
	feign.compression.response.enabled=true

Feign request compression gives you settings similar to what you may set for your web server:

Feign提供的压缩设置与你的Web server的设置类似：

	
	feign.compression.request.enabled=true
	feign.compression.request.mime-types=text/xml,application/xml,application/json
	feign.compression.request.min-request-size=2048

These properties allow you to be selective about the compressed media types and minimum request threshold length.

这些属性允许你选择要压缩的 MIME-TYPE 和最小的请求长度。

### Feign logging Feign 日志
A logger is created for each Feign client created. By default the name of the logger is the full class name of the interface used to create the Feign client. Feign logging only responds to the DEBUG level.

每个Feign client都创建了一个logger。默认的logger的命名是Feign client的全限定名。Feign日志只响应 DEBUG 级别。

application.yml

	logging.level.project.user.UserClient: DEBUG

The Logger.Level object that you may configure per client, tells Feign how much to log. Choices are:

- NONE, No logging (DEFAULT).
- BASIC, Log only the request method and URL and the response status code and execution time.
- HEADERS, Log the basic information along with request and response headers.
- FULL, Log the headers, body, and metadata for both requests and responses.

你能为每个客户端配置Logger.Level 对象，告诉Feign记录多少日志,选项包括: 

- NONE, 不记录 (DEFAULT). 
- BASIC, 仅记录请求方式和URL及响应的状态代码与执行时间. 
- HEADERS, 日志的基本信息与请求及响应的头. 
- FULL, 记录请求与响应的头和正文及元数据.

For example, the following would set the Logger.Level to FULL:

例如，下面的设置会让 Logger.Level为FULL.

	@Configuration
	public class FooConfiguration {
	    @Bean
	    Logger.Level feignLoggerLevel() {
	        return Logger.Level.FULL;
	    }
	}


### External Configuration: Archaius
Archaius is the Netflix client side configuration library. It is the library used by all of the Netflix OSS components for configuration. Archaius is an extension of the Apache Commons Configuration project. It allows updates to configuration by either polling a source for changes or for a source to push changes to the client. Archaius uses Dynamic<Type>Property classes as handles to properties.

Archaius是Netflix client端配置库。它的配置可以被所有的Netflix OSS组件使用。Archaius是 Apache Commons Configuration 的项目。它允许更新配置通过轮询或者推送到client的方式。Archaius使用动态属性类属性的处理属性。

Archaius Example

	class ArchaiusTest {
	    DynamicStringProperty myprop = DynamicPropertyFactory
	            .getInstance()
	            .getStringProperty("my.prop");
	
	    void doSomething() {
	        OtherClass.someMethod(myprop.get());
	    }
	}

Archaius has its own set of configuration files and loading priorities. Spring applications should generally not use Archaius directly, but the need to configure the Netflix tools natively remains. Spring Cloud has a Spring Environment Bridge so Archaius can read properties from the Spring Environment. This allows Spring Boot projects to use the normal configuration toolchain, while allowing them to configure the Netflix tools, for the most part, as documented.

Archaius有它自己的一套配置文件和负载优先级， Spring 应用程序通常不应直接应用Archaius, 本身仍然有配置Netflix工具的需求。Spring Cloud有一个Spring Environment Bridge，所以Archaius可以通过spring environment读取属性。这允许spring boot项目使用配置工具链，while allowing them to configure the Netflix tools, for the most part, as documented.
