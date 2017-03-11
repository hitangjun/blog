---
title: Spring Cloud Netflix--Zuul Guide
date: 2017-03-11 10:55:18
description: 
categories:
- Spring Cloud Netflix
tags:
- Zuul Guide
toc: true
author: John Tang
comments:
original:
permalink: 
---

### Zuul Developer Guide Zuul开发指南
For a general overview of how Zuul works, please see the Zuul Wiki.

#### The Zuul Servlet
Zuul is implemented as a Servlet. For the general cases, Zuul is embedded into the Spring Dispatch mechanism. This allows Spring MVC to be in control of the routing. In this case, Zuul is configured to buffer requests. If there is a need to go through Zuul without buffering requests (e.g. for large file uploads), the Servlet is also installed outside of the Spring Dispatcher. By default, this is located at /zuul. This path can be changed with the zuul.servlet-path property.
<!--more-->

#### Zuul RequestContext
To pass information between filters, Zuul uses a RequestContext. Its data is held in a ThreadLocal specific to each request. Information about where to route requests, errors and the actual HttpServletRequest and HttpServletResponse are stored there. The RequestContext extends ConcurrentHashMap, so anything can be stored in the context. FilterConstants contains the keys that are used by the filters installed by Spring Cloud Netflix (more on these later).

#### @EnableZuulProxy vs. @EnableZuulServer
Spring Cloud Netflix installs a number of filters based on which annotation was used to enable Zuul. @EnableZuulProxy is a superset of @EnableZuulServer. In other words, @EnableZuulProxy contains all filters installed by @EnableZuulServer. The additional filters in the "proxy" enable routing functionality. If you want a "blank" Zuul, you should use @EnableZuulServer.

#### @EnableZuulServer Filters
Creates a SimpleRouteLocator that loads route definitions from Spring Boot configuration files.

The following filters are installed (as normal Spring Beans):

Pre filters:

- ServletDetectionFilter: Detects if the request is through the Spring Dispatcher. Sets boolean with key FilterConstants.IS_DISPATCHER_SERVLET_REQUEST_KEY.
- FormBodyWrapperFilter: Parses form data and reencodes it for downstream requests.
- DebugFilter: if the debug request parameter is set, this filter sets RequestContext.setDebugRouting() and RequestContext.setDebugRequest() to true.

Route filters:

- SendForwardFilter: This filter forwards requests using the Servlet RequestDispatcher. The forwarding location is stored in the RequestContext attribute FilterConstants.FORWARD_TO_KEY. This is useful for forwarding to endpoints in the current application.

Post filters:

- SendResponseFilter: Writes responses from proxied requests to the current response.

Error filters:

- SendErrorFilter: Forwards to /error (by default) if RequestContext.getThrowable() is not null. The default forwarding path (/error) can be changed by setting the error.path property.

#### @EnableZuulProxy Filters
Creates a DiscoveryClientRouteLocator that loads route definitions from a DiscoveryClient (like Eureka), as well as from properties. A route is created for each serviceId from the DiscoveryClient. As new services are added, the routes will be refreshed.

In addition to the filters described above, the following filters are installed (as normal Spring Beans):

Pre filters:

- PreDecorationFilter: This filter determines where and how to route based on the supplied RouteLocator. It also sets various proxy-related headers for downstream requests.

Route filters:


- RibbonRoutingFilter: This filter uses Ribbon, Hystrix and pluggable HTTP clients to send requests. Service ids are found in the RequestContext attribute FilterConstants.SERVICE_ID_KEY. This filter can use different HTTP clients. They are:
- Apache HttpClient. This is the default client.
- Squareup OkHttpClient v3. This is enabled by having the com.squareup.okhttp3:okhttp library on the classpath and setting ribbon.okhttp.enabled=true.
- Netflix Ribbon HTTP client. This is enabled by setting ribbon.restclient.enabled=true. This client has limitations, such as it doesn’t support the PATCH method, but also has built-in retry.
- SimpleHostRoutingFilter: This filter sends requests to predetermined URLs via an Apache HttpClient. URLs are found in RequestContext.getRouteHost().

### How to Write a Pre Filter
Pre filters are used to set up data in the RequestContext for use in filters downstream. The main use case is to set information required for route filters.

	public class QueryParamPreFilter extends ZuulFilter {
		@Override
		public int filterOrder() {
			return PRE_DECORATION_FILTER_ORDER - 1; // run before PreDecoration
		}
	
		@Override
		public String filterType() {
			return PRE_TYPE;
		}
	
		@Override
		public boolean shouldFilter() {
			RequestContext ctx = RequestContext.getCurrentContext();
			return !ctx.containsKey(FORWARD_TO_KEY) // a filter has already forwarded
					&& !ctx.containsKey(SERVICE_ID_KEY); // a filter has already determined serviceId
		}
	    @Override
	    public Object run() {
	        RequestContext ctx = RequestContext.getCurrentContext();
			HttpServletRequest request = ctx.getRequest();
			if (request.getParameter("foo") != null) {
			    // put the serviceId in `RequestContext`
	    		ctx.put(SERVICE_ID_KEY, request.getParameter("foo"));
	    	}
	        return null;
	    }
	}
The filter above populates SERVICE_ID_KEY from the foo request parameter. In reality, it’s not a good idea to do that kind of direct mapping, but the service id should be looked up from the value of foo instead.

Now that SERVICE_ID_KEY is populated, PreDecorationFilter won’t run and RibbonRoutingFilter will. If you wanted to route to a full URL instead, call ctx.setRouteHost(url) instead.

To modify the path that routing filters will forward to, set the REQUEST_URI_KEY.

### How to Write a Route Filter
Route filters are run after pre filters and are used to make requests to other services. Much of the work here is to translate request and response data to and from the client required model.

	public class OkHttpRoutingFilter extends ZuulFilter {
		@Autowired
		private ProxyRequestHelper helper;
	
		@Override
		public String filterType() {
			return ROUTE_TYPE;
		}
	
		@Override
		public int filterOrder() {
			return SIMPLE_HOST_ROUTING_FILTER_ORDER - 1;
		}
	
		@Override
		public boolean shouldFilter() {
			return RequestContext.getCurrentContext().getRouteHost() != null
					&& RequestContext.getCurrentContext().sendZuulResponse();
		}
	
	    @Override
	    public Object run() {
			OkHttpClient httpClient = new OkHttpClient.Builder()
					// customize
					.build();
	
			RequestContext context = RequestContext.getCurrentContext();
			HttpServletRequest request = context.getRequest();
	
			String method = request.getMethod();
	
			String uri = this.helper.buildZuulRequestURI(request);
	
			Headers.Builder headers = new Headers.Builder();
			Enumeration<String> headerNames = request.getHeaderNames();
			while (headerNames.hasMoreElements()) {
				String name = headerNames.nextElement();
				Enumeration<String> values = request.getHeaders(name);
	
				while (values.hasMoreElements()) {
					String value = values.nextElement();
					headers.add(name, value);
				}
			}
	
			InputStream inputStream = request.getInputStream();
	
			RequestBody requestBody = null;
			if (inputStream != null && HttpMethod.permitsRequestBody(method)) {
				MediaType mediaType = null;
				if (headers.get("Content-Type") != null) {
					mediaType = MediaType.parse(headers.get("Content-Type"));
				}
				requestBody = RequestBody.create(mediaType, StreamUtils.copyToByteArray(inputStream));
			}
	
			Request.Builder builder = new Request.Builder()
					.headers(headers.build())
					.url(uri)
					.method(method, requestBody);
	
			Response response = httpClient.newCall(builder.build()).execute();
	
			LinkedMultiValueMap<String, String> responseHeaders = new LinkedMultiValueMap<>();
	
			for (Map.Entry<String, List<String>> entry : response.headers().toMultimap().entrySet()) {
				responseHeaders.put(entry.getKey(), entry.getValue());
			}
	
			this.helper.setResponse(response.code(), response.body().byteStream(),
					responseHeaders);
			context.setRouteHost(null); // prevent SimpleHostRoutingFilter from running
			return null;
	    }
	}
The above filter translates Servlet request information into OkHttp3 request information, executes an HTTP request, then translates OkHttp3 reponse information to the Servlet response. WARNING: this filter might have bugs and not function correctly.

#### How to Write a Post Filter
Post filters typically manipulate the response. In the filter below, we add a random UUID as the X-Foo header. Other manipulations, such as transforming the response body, are much more complex and compute-intensive.

	public class AddResponseHeaderFilter extends ZuulFilter {
		@Override
		public String filterType() {
			return POST_TYPE;
		}
	
		@Override
		public int filterOrder() {
			return SEND_RESPONSE_FILTER_ORDER - 1;
		}
	
		@Override
		public boolean shouldFilter() {
			return true;
		}
	
		@Override
		public Object run() {
			RequestContext context = RequestContext.getCurrentContext();
	    	HttpServletResponse servletResponse = context.getResponse();
			servletResponse.addHeader("X-Foo", UUID.randomUUID().toString());
			return null;
		}
	}
