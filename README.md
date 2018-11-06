API服务网关Zuul。

微服务场景下，每一个微服务对外暴露了一组细粒度的服务。客户端的请求可能会涉及到一串的服务调用，如果将这些微服务都暴露给客户端，那么客户端需要多次请求不同的微服务才能完成一次业务处理，增加客户端的代码复杂度。另外，对于微服务我们可能还需要服务调用进行统一的认证和校验等等。微服务架构虽然可以将我们的开发单元拆分的更细，降低了开发难度，但是如果不能够有效的处理上面提到的问题，可能会造成微服务架构实施的失败。

Zuul参考GOF设计模式中的Facade模式，将细粒度的服务组合起来提供一个粗粒度的服务，所有请求都导入一个统一的入口，那么整个服务只需要暴露一个api，对外屏蔽了服务端的实现细节，也减少了客户端与服务器的网络调用次数。这就是API服务网关(API Gateway)服务。我们可以把API服务网关理解为介于客户端和服务器端的中间层，所有的外部请求都会先经过API服务网关。因此，API服务网关几乎成为实施微服务架构时必须选择的一环。

Spring Cloud Netflix的Zuul组件可以做反向代理的功能，通过路由寻址将请求转发到后端的粗粒度服务上，并做一些通用的逻辑处理。

通过Zuul我们可以完成以下功能：

* 动态路由
* 监控与审查
* 身份认证与安全
* 压力测试: 逐渐增加某一个服务集群的流量，以了解服务性能;
* 金丝雀测试
* 服务迁移
* 负载剪裁: 为每一个负载类型分配对应的容量，对超过限定值的请求弃用;
* 静态应答处理

> 构建Zuul-Server

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```
`@EnableZuulProxy`启动Zuul的路由服务。

然后是配置文件。

```
server.port=8280
spring.application.name=ZUUL-PROXY
eureka.client.service-url.defaultZone=http://localhost:8081/eureka
```

然后再创建一个微服务。`BusEndPoint`

然后依次启动`Eureka Server`,`所有的服务`,`Zuul Server`。

然后在浏览器中输入:`http://localhost:8180/user-service/users/1`和`http://localhost:8180/bus-service/buses`来调用服务。

这样负载均衡和容错仍然起作用。


> Zuul配置。

路由配置详解。

1） 默认配置

`http://localhost:zuul-port/service-name/rest-api`如:`http://localhost:8180/bus-service/buses`

2) 自定义微服务访问路径

**zuul.routes.微服务Id = 指定路径**如:`zuul.routes.user-service = /user/**`。

/user/**能匹配多有/user开头的路径。

3）忽略指定微服务

 `zuul.ignored-services=微服务Id1,微服务Id2`如:`zuul.ignored-services=user-service,product-service`。
 
4）同时指定微服务id和对应路径。

```
zuul.routes.api-a.path=/api-a/**
zuul.routes.api-a.serviceId=service-A

zuul.routes.api-b.path=/api-b/**
zuul.routes.api-b.serviceId=service-B
```

5）同时指定微服务Url和对应路径。

```
zuul.routes.api-a.path=/api-a/**
zuul.routes.api-a.url=http://localhost:8080/api-a
```
6)指定多个服务实例及负载均衡

```
zuul.routes.user.path: /user/**
zuul.routes.user.serviceId: user

ribbon.eureka.enabled=false
user.ribbon.listOfServers: http://192.168.1.10:8081, http://192.168.1.11:8081
```

7)forward跳转到本地url

```
zuul.routes.user.path=/user/**
zuul.routes.user.url=forward:/user
```
8)路由前缀
可以通过zuul.prefix可为所有的映射增加统一的前缀。如: /api。默认情况下，代理会在转发前自动剥离这个前缀。如果需要转发时带上前缀，可以配置: zuul.stripPrefix=false来关闭这个默认行为。例如：

```
zuul.routes.users.path=/myusers/**
zuul.routes.users.stripPrefix=false
```

注:zuul.stripPrefix只会对zuul.prefix的前缀起作用。对于path指定的前缀不会起作用。

9)路由配置顺序

如果想按照配置的顺序进行路由规则控制，则需要使用YAML，如果是使用propeties文件，则会丢失顺序。例如:

```
zuul:
  routes:
    users:
      path: /myusers/**
    legacy:
      path: /**
```
上例如果是使用properties文件进行配置，则legacy就可能会先生效，这样users就没效果了。

10)自定义转换

我们也可以一个转换器，让serviceId和路由之间使用正则表达式来自动匹配。例如：

```
@Bean
public PatternServiceRouteMapper serviceRouteMapper() {
    return new PatternServiceRouteMapper(
        "(?<name>^.+)-(?<version>v.+$)",
        "${version}/${name}");
}
```    
这样，serviceId为“users-v1”的服务，就会被映射到路由为“/v1/users/”的路径上。任何正则表达式都可以，但是所有的命名组必须包括servicePattern和routePattern两部分。如果servicePattern没有匹配一个serviceId，那就会使用默认的。在上例中，一个serviceId为“users”的服务，将会被映射到路由“/users/”中（不带版本信息）。这个特性默认是关闭的，而且只适用于已经发现的服务。

> Zuul Header设置

敏感Header设置。

同一个系统中各个服务之间通过Headers来共享信息是没啥问题的，但是如果不想Headers中的一些敏感信息随着HTTP转发泄露出去话，需要在路由配置中指定一个忽略Header的清单。  

默认情况下，Zuul在请求路由时，会过滤HTTP请求头信息中的一些敏感信息，默认的敏感头信息通过zuul.sensitiveHeaders定义，包括Cookie、Set-Cookie、Authorization。配置的sensitiveHeaders可以用逗号分割。

对指定路由下的进行配置：

```
# 对指定路由开启自定义敏感头
zuul.routes.[route].customSensitiveHeaders=true 
zuul.routes.[route].sensitiveHeaders=[这里设置要过滤的敏感头]
```
设置全局:

`zuul.sensitiveHeaders=[这里设置要过滤的敏感头]`

忽略Header设置：

如果每一个路由都需要配置一些额外的敏感Header时，那你可以通过zuul.ignoredHeaders来统一设置需要忽略的Header。如:
 
 `zuul.ignoredHeaders=[这里设置要忽略的Header]`
 
在默认情况下是没有这个配置的，如果项目中引入了Spring Security，那么Spring Security会自动加上这个配置，默认值为: Pragma,Cache-Control,X-Frame-Options,X-Content-Type-Options,X-XSS-Protection,Expries。

此时，如果还需要使用下游微服务的Spring Security的Header时，可以增加下面的设置:

`zuul.ignoreSecurityHeaders=false`

> Zuul Http Client

Zuul的Http客户端支持Apache Http、Ribbon的RestClient和OkHttpClient，默认使用Apache HTTP客户端。可以通过下面的方式启用相应的客户端：

```
# 启用Ribbon的RestClient
ribbon.restclient.enabled=true

# 启用OkHttpClient
ribbon.okhttp.enabled=true
```
如果需要启用OKHttpClient,需要注意在你的项目中已经包含`com.squareup.okhttp3`相关包。

> Zuul 容错和回退。

请注意，Zuul的Hystrix监控的粒度是微服务，而不是某个API，也就是所有经过Zuul的请求都会被Hystrix保护起来。假如，我们现在把Product-Service服务关闭，再来访问会出现什么结果呢？结果可能不是我们所想那样，如下：

![](https://upload-images.jianshu.io/upload_images/1488771-2596a0ad3d857d60.png?imageMogr2/auto-orient/)

呃，比较郁闷是么！那么如何为Zuul实现容错与回退呢？

Zuul提供了一个`ZuulFallbackProvider`接口，通过实现该接口就可以为Zuul实现回退功能。那么让我们改造之前的Zuul-Server。

代码如下。

```
@Component
public class UserServiceFallbackProvider implements FallbackProvider {
    @Override
    public String getRoute() {
        // 注意: 这里是route的名称，不是服务的名称，
        // 如果这里写成大写USER-SERVICE将无法起到回退作用
        return "user-service";
    }

    @Override
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
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
                return "ok";
            }

            @Override
            public void close() {

            }

            @Override
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream("商品服务暂时不可用，请稍后重试!".getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON_UTF8);
                return headers;
            }
        };
    }
}
```
重启，然后你会看到。

![](https://upload-images.jianshu.io/upload_images/1488771-597f37793a9155bd.png?imageMogr2/auto-orient/)



Zuul给我们的第一印象通常是这样：它包含了对请求的路由和过滤两个功能，其中路由功能负责将外部请求转发到具体的微服务实例上，是实现外部访问统一入口的基础。过滤器功能则负责对请求的处理过程进行干预，是实现请求校验、服务聚合等功能的基础。然而实际上，路由功能在真正运行时，它的路由映射和请求转发都是由几个不同的过滤器完成的。其中，路由映射主要是通过PRE类型的过滤器完成，它将请求路径与配置的路由规则进行匹配，以找到需要转发的目标地址。而请求转发的部分则是由Route类型的过滤器来完成，对PRE类型过滤器获得的路由地址进行转发。所以，过滤器可以说是Zuul实现API网关功能最重要的核心部件，每一个进入Zuul的请求都会经过一系列的过滤器处理链得到请求响应并返回给客户端。

> 过滤器

Zuul过滤器的关键特性有:

* Type: 定义在请求执行过程中何时被执行;
* Execution Order: 当存在多个过滤器时，用来指示执行的顺序，值越小就会越早执行;
* Criteria: 执行的条件，即该过滤器何时会被触发;
* Action: 具体的动作。

过滤器之间并不会直接进行通信，而是通过RequestContext来共享信息，RequestContext是线程安全的。

对应上面Zuul过滤器的特性，我们在实现一个自定义过滤器时需要实现的方法有：

```
public class PreTypeZuulFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "PRE_TYPE";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        return null;
    }
}
```
其中:

* filterType()方法是该过滤器的类型;
* filterOrder()方法返回的是执行顺序;
* shouldFilter()方法则是判断是否需要执行该过滤器;
* run()则是所要执行的具体过滤动作。

过滤器的类型有:

Zuul中定义了四种标准的过滤器类型，这些过滤器类型对应于请求的典型生命周期。

* PRE过滤器: 在请求被路由之前调用, 可用来实现身份验证、在集群中选择请求的微服务、记录调试信息等;
* ROUTING过滤器: 在路由请求时候被调用;
* POST过滤器: 在路由到微服务以后执行, 可用来为响应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等;
* ERROR过滤器: 在处理请求过程时发生错误时被调用。

Zuul过滤器的类型其实也是Zuul过滤器的生命周期，通过下面这张图来了解它们的执行过程。

![](https://upload-images.jianshu.io/upload_images/1488771-f9b362aa2f98c460.png?imageMogr2/auto-orient/)

除了上面给出的四种默认的过滤器类型之外，Zuul还允许我们创建自定义的过滤器类型。例如，我们可以定制一种STATIC类型的过滤器，直接在Zuul中生成响应，而不将请求转发到后端的微服务。

自定义过滤器代码:

```
package com.lanyage.zuul.zuulserver.filter;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.netflix.zuul.exception.ZuulException;

import javax.servlet.http.HttpServletRequest;

public class QueryParamPreFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "PRE_TYPE";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext requestContext = RequestContext.getCurrentContext();
        return !requestContext.containsKey("FORWARD_TO_KEY")
                && !requestContext.containsKey("SERVICE_ID_KEY");
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        if (request.getParameter("foo") != null) {
            ctx.put("SERVICE_ID_KEY", request.getParameter("foo"));
        }
        return null;
    }
}
```

这个是官方给出的一个示例，从请求的参数foo中获取需要转发到的服务Id。当然官方并不建议我们这么做，这里只是方便给出一个示例而已。

Route类型过滤器。

```
package com.lanyage.zuul.zuulserver.filter;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.netflix.zuul.exception.ZuulException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.netflix.zuul.filters.ProxyRequestHelper;

public class OkHttpRoutingFilter extends ZuulFilter {

    private static final Integer SIMPLE_HOST_ROUTING_FILTER_ORDER = 2;
    @Autowired
    private ProxyRequestHelper helper;
    @Override
    public String filterType() {
        return "ROUTE_TYPE";
    }

    @Override
    public int filterOrder() {
        return SIMPLE_HOST_ROUTING_FILTER_ORDER - 1;
    }

    @Override
    public boolean shouldFilter() {
        return RequestContext.getCurrentContext().getRouteHost() != null &&
                RequestContext.getCurrentContext().sendZuulResponse();
    }

    @Override
    public Object run() throws ZuulException {
//       OkHttpClient httpClient = new OkHttpClient.Builder().build();
//
//        RequestContext context = RequestContext.getCurrentContext();
//        HttpServletRequest request = context.getRequest();
//        String method = request.getMethod();
//        String uri = this.helper.buildZuulRequestURI(request);
//        Headers.Builder headers = new Headers.Builder();
//        Enumeration<String> headerNames = request.getHeaderNames();
//        while (headerNames.hasMoreElements()) {
//            String name = headerNames.nextElement();
//            Enumeration<String> values = request.getHeaders(name);
//
//            while (values.hasMoreElements()) {
//                String value = values.nextElement();
//                headers.add(name, value);
//            }
//        }
//        InputStream inputStream = request.getInputStream();
//        RequestBody requestBody = null;
//        if (inputStream != null && HttpMethod.permitsRequestBody(method)) {
//            MediaType mediaType = null;
//            if (headers.get("Content-Type") != null) {
//                mediaType = MediaType.parse(headers.get("Content-Type"));
//            }
//            requestBody = RequestBody.create(mediaType, StreamUtils.copyToByteArray(inputStream));
//        }
//        Request.Builder builder = new Request.Builder()
//                .headers(headers.build())
//                .url(uri)
//                .method(method, requestBody);
//
//        Response response = httpClient.newCall(builder.build()).execute();
//
//        LinkedMultiValueMap<String, String> responseHeaders = new LinkedMultiValueMap<>();
//        for (Map.Entry<String, List<String>> entry : response.headers().toMultimap().entrySet()) {
//            responseHeaders.put(entry.getKey(), entry.getValue());
//        }
//
//        this.helper.setResponse(response.code(), response.body().byteStream(),          responseHeaders);
//        context.setRouteHost(null); // prevent SimpleHostRoutingFilter from running
        return null;
    }
}
```
这个示例是将HTTP请求转换为使用OkHttp3进行请求，并将服务端的返回转换成Servlet的响应。

**注意: 官方说这仅仅是一个示例，功能不一定正确。**

POST类型示例。

```
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
        HttpServletResponse servletResponse = context.getResponse();        servletResponse.addHeader("X-Foo", UUID.randomUUID().toString()); 
        return null;
    }
}
```
这个示例很简单就是返回的头中增加一个随机生成X-Foo。

禁用过滤器。

只需要在application.properties(或yml)中配置需要禁用的filter，格式为:zuul.[filter-name].[filter-type].disable=true。如:

`zuul.FormBodyWrapperFilter.pre.disable=true`

关于Zuul过滤器Error的一点补充。

当Zuul在执行过程中抛出一个异常时，error过滤器就会被执行。而SendErrorFilter只有在RequestContext.getThrowable()不为空的时候才会执行。它将错误信息设置到请求的javax.servlet.error.*属性中，并转发Spring Boot的错误页面。

Zuul过滤器实现的具体类是`ZuulServletFilter`，其核心代码如下:

```
@Override
public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
    try {
        init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
        try {
            preRouting();
        } catch (ZuulException e) {
            error(e);
            postRouting();
            return;
        }
        
        // Only forward onto to the chain if a zuul response is not being sent
        if (!RequestContext.getCurrentContext().sendZuulResponse()) {
            filterChain.doFilter(servletRequest, servletResponse);
            return;
        }
        
        try {
            routing();
        } catch (ZuulException e) {
            error(e);
            postRouting();
            return;
        }
        try {
            postRouting();
        } catch (ZuulException e) {
            error(e);
            return;
        }
    } catch (Throwable e) {
        error(new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_FROM_FILTER_" + e.getClass().getName()));
    } finally {
        RequestContext.getCurrentContext().unset();
    }
}
```

从这段代码中可以看出，error可以在所有阶段捕获异常后执行，但是如果post阶段中出现异常被error处理后则不再回到post阶段执行，也就是说需要保证在post阶段不要有异常，因为一旦有异常后就会造成该过滤器后面其它post过滤器将不再被执行。

一个简单的全局异常处理的方法是: 添加一个类型为error的过滤器，将错误信息写入RequestContext，这样SendErrorFilter就可以获取错误信息了。代码如下:

```
public class GlobalErrorFilter extends ZuulFilter { 
    @Override
    public String filterType() { 
        return ERROR_TYPE;
    }
    
    @Override
    public int filterOrder() {
        return 10; 
    }

    @Override
    public boolean shouldFilter() {
        return true; 
    }

    @Override
    public Object run() {
        RequestContext context = RequestContext.getCurrentContext();
        Throwable throwable = context.getThrowable();
        this.logger.error("[ErrorFilter] error message: {}", throwable.getCause().getMessage());
        context.set("error.status_code", HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
        context.set("error.exception", throwable.getCause());
        return null;
    }
}
```

`@EnableZuulServer` VS. `@EnableZuulProxy`

Zuul为我们提供了两个主应用注解: @EnableZuulServer和@EnableZuulProxy，其中@EnableZuulProxy包含@EnableZuulServer的功能，而且还加入了@EnableCircuitBreaker和@EnableDiscoveryClient。当我们需要运行一个没有代理功能的Zuul服务，或者有选择的开关部分代理功能时，那么需要使用 @EnableZuulServer 替代 @EnableZuulProxy。 这时候我们可以添加任何 ZuulFilter类型实体类都会被自动加载，这和上一篇使用@EnableZuulProxy是一样，但不会自动加载任何代理过滤器。

> `@EnableZuulServer`默认加载的所有过滤器。

* PRE类型过滤器 
	
	* ServletDetectionFilter。该过滤器是最先被执行的。其主要用来检查当前请求是通过Spring的DispatcherServlet处理运行的，还是通过ZuulServlet来处理运行的。判断结果会保存在isDispatcherServletRequest中，值类型为布尔型。
	* FormBodyWrapperFilter。该过滤器的目的是将符合要求的请求体包装成FormBodyRequestWrapper对象，以供后续处理使用。
	* DebugFilter。当请求参数中设置了debug参数时，该过滤器会将当前请求上下文中的RequestContext.setDebugRouting()和RequestContext.setDebugRequest()设置为true，这样后续的过滤器可以根据这两个参数信息定义一些debug信息，当生产环境出现问题时，我们就可以通过增加该参数让后台打印出debug信息，以帮助我们进行问题分析。对于请求中的debug参数的名称，我们可以通过zuul.debug.parameter进行自定义。
	
* ROUTE类型过滤器
	* SendForwardFilter。该过滤器只对请求上下文中存在forward.to(FilterConstants.FORWARD_TO_KEY)参数的请求进行处理。即处理之前我们路由规则中forward的本地跳转。
	
* POST类型过滤器
	* SendResponseFilter。该过滤器就是对代理请求所返回的响应进行封装，然后作为本次请求的相应发送回给请求者。
* Error类型过滤器
	* SendErrorFilter。该过滤器就是判断当前请求上下文中是否有异常信息(RequestContext.getThrowable()不为空)，如果有则默认转发到/error页面，我们也可以通过设置error.path来自定义错误页面。 


> `@EnableZuulProxy`默认过滤器		  

* PRE类型过滤器
	* PreDecorationFilter。该过滤器根据提供的RouteLocator确定路由到的地址，以及怎样去路由。该路由器也可为后端请求设置各种代理相关的header。
* Route类型过滤器
	* RibbonRoutingFilter。该过滤器会针对上下文中存在serviceId(可以通过`RequestContext.getCurrentContext().get(“serviceId”)`获取)的请求进行处理，使用Ribbon、Hystrix和可插拔的HTTP客户端发送请求，并将服务实例的请求结果返回。也就是之前所说的只有当我们使用serviceId配置路由规则时Ribbon和Hystrix方才生效。
	* SimpleHostRoutingFilter。该过滤器检测到routeHost参数(可通过RequestContext.getRouteHost()获取)设置时，就会通过Apache HttpClient向指定的URL发送请求。此时，请求不会使用Hystrix命令进行包装，所以这类请求也就没有线程隔离和断路器保护。



  