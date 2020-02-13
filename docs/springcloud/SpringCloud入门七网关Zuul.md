一、简介

网关是微服务应用中必不可少的一个环节，有同学会问我已经有了nginx为什么还需要程序网关呢？

- nginx可以做到负载均衡、路由转发 但是在当有多个复杂应用时往往会各自需要一个统一访问入口、统一认证、鉴权等业务功能

- 日志记录，可以在网关层实现统一的日志记录

- 安全校验，增加各类接口安全排查等

  

Zuul是Netflix的基于JVM的路由器和服务器端负载均衡器。

Zuul 目前有两个大的版本，1.x 和 2.x，这两个版本差别很大。

- 在1.x中是基于同步 IO做请求转发 核心功能是pre、routing、post 三个过滤器 

- 而到了2.x改成了基于Netty Server 实现了异步 IO 来接入请求，同时过滤器名已经变成了Inbound Filter、Endpoint Filter 和 Outbound Filter

另外，springcloud也自研了一个网关 spring-cloud-gateway ，在性能上与zuul2相差不大，大概是直连的40%，使用起来各有特色。

值得注意的是scg可以实现websocket 转发，但是不能使用shiro ，而zuul不能使用websocket 但是却可以很好的接入shiro  所以在使用选型的时候大家可以各取所需

在spring cloud 的Greenwich版本中使用zuul2.x版本进行演示 后面有机会会再研究scg的实现

二、网关架构

使用网关后的微服务架构

<p align="center">
    <a href="https://tva1.sinaimg.cn/large/0082zybpgy1gbupod91fyj30wa0u0dj9.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbupod91fyj30wa0u0dj9.jpg" width=""/>
    </a>
</p> 

三、pom

```
<dependencies>
    <!-- spring boot web依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- 服务注册 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    <!-- 监控神器 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- zuul -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
</dependencies>
```

四、配置文件

zuul的路由配置有很多方式，在这里简单介绍三个

- 本地服务的接口路径转发
- 注册的微服务通过serverId转发
- 未注册的服务直接配置相应的路径进行转发

请求的时候

​     访问路径= 网关地址/路由地址/请求地址?请求参数

比如原来请求地址为 http://127.0.0.1:8081/demo?name=test

加了网关之后 http://网关host:port/配置的路由/demo?name=test

```
#服务名称
spring:
  application:
    name: zuul-server
#服务启动端口
server:
  port: 10095
logging:
  config: classpath:logback.xml


#主机地址
eureka:
  instance:
    prefer-ip-address: true #优先使用ip地址的方式注册
    appname: ${spring.application.name}
    instance-id: ${spring.application.name}@${spring.cloud.client.ip-address}:${server.port}
  client:
    healthcheck:
      enabled: true #服务健康检查
    #服务地址
    service-url:
      defaultZone:  http://127.0.0.1:10086/eureka/


zuul:
  ignoredServices: '*' #只暴露配置了路由的服务 不配置会暴露注册中心其他的服务
  sensitive-headers: Access-Control-Allow-Origin,Access-Control-Allow-Methods,Access-Control-Allow-Credentials,Access-Control-Allow-Headers,Access-Control-Expose-Headers,Access-Control-Max-Age
  routes:
    #本地服务
    localTest:
      path: /test/**
      url: forward:/testDemo
    #注册的微服务
    equip-service:
      path: /producer-server/**
      serviceId: producer-server
    #自定义路由(可以访问未注册的外部服务)
    demo:
      path: /demo-producer-server/**
      url: http://127.0.0.1:8082 #自定义路由路径
  host:
    connect-timeout-millis: 10000 #url方式的超时设置
    socket-timeout-millis: 3000
```

五、测试代码

```
package com.neo.zuul.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * <p></p>
 *
 * @author neo
 * @date 2020/2/11 17:17
 */
@RestController
public class ZuulTestController {

    @RequestMapping(value = "/testDemo")
    public String test(@RequestParam String name) {
        return "localhost server , hello " + name;
    }
}
```

```
@Autowired
RestTemplate restTemplate;

@Test
public void testZuul() {
    String name = "zuul";
    String res = restTemplate.getForObject("http://zuul-server/test?name=" + name, String.class);
    log.info("网关本地路径转发访问:"+res);
    res = restTemplate.getForObject("http://zuul-server/producer-server/hello?name=" + name, String.class);
    log.info("根据serverId转发访问:"+res);
    res = restTemplate.getForObject("http://zuul-server/demo-producer-server/hello?name=" + name, String.class);
    log.info("自定义路径转发访问:"+res);
}
```

六、zuul+ribbon+hystrix+feign 超时配置

如果在使用zuul时也使用了hystrix，此时需要注意一点,

zuul+ribbon+hystrix+feign都有各自配置的请求超时时间，

配置举例如下：

```
ribbon:
  ReadTimeout:  1000
  ConnectTimeout:  300
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 15000 #该时间会对feigin生效
      circuitBreaker:
        sleepWindowInMilliseconds: 10s #短路多久以后开始尝试是否恢复，默认5s
zuul:
  host:
    connect-timeout-millis: 1000 #url方式的超时设置
    socket-timeout-millis: 300
```

zuul 中配置超时时间有两种配置方式

- 用 serviceId 进行路由时，使用 `ribbon.ReadTimeout` 和 `ribbon.SocketTimeout` 设置
- 用指定 url 进行路由时，使用 `zuul.host.connect-timeout-millis` 和 `zuul.host.socket-timeout-millis` 设置

具体服务的hytrix超时时间 > 默认的hytrix超时时间 > ribbon超时时间

这个地方我们可以直接看源码 AbstractRibbonCommand.java

### hytrix超时时间

```java
protected static int getHystrixTimeout(IClientConfig config, String commandKey) {
   int ribbonTimeout = getRibbonTimeout(config, commandKey);
   DynamicPropertyFactory dynamicPropertyFactory = DynamicPropertyFactory.getInstance();
  // 获取默认超时时间
   int defaultHystrixTimeout = dynamicPropertyFactory.getIntProperty("hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds",
      0).get();
  //获取具体服务的hytrix超时时间，这里应该是hystrix.command.foo.execution.isolation.thread.timeoutInMilliseconds
   int commandHystrixTimeout = dynamicPropertyFactory.getIntProperty("hystrix.command." + commandKey + ".execution.isolation.thread.timeoutInMilliseconds",
      0).get();
   int hystrixTimeout;
  // hystrixTimeout的优先级是 具体服务的hytrix超时时间 > 默认的hytrix超时时间 > ribbon超时时间
   if(commandHystrixTimeout > 0) {
      hystrixTimeout = commandHystrixTimeout;
   }
   else if(defaultHystrixTimeout > 0) {
      hystrixTimeout = defaultHystrixTimeout;
   } else {
      hystrixTimeout = ribbonTimeout;
   }
  // 如果默认的或者具体服务的hytrix超时时间小于ribbon超时时间就会警告
   if(hystrixTimeout < ribbonTimeout) {
      LOGGER.warn("The Hystrix timeout of " + hystrixTimeout + "ms for the command " + commandKey +
         " is set lower than the combination of the Ribbon read and connect timeout, " + ribbonTimeout + "ms.");
   }
   return hystrixTimeout;
}
```
### ribbon超时时间

```java
protected static int getRibbonTimeout(IClientConfig config, String commandKey) {
   int ribbonTimeout;
   if (config == null) {
      ribbonTimeout = RibbonClientConfiguration.DEFAULT_READ_TIMEOUT + RibbonClientConfiguration.DEFAULT_CONNECT_TIMEOUT;
   } else {
     	   // 这里获取了四个参数，ReadTimeout，ConnectTimeout，MaxAutoRetries， MaxAutoRetriesNextServer

      int ribbonReadTimeout = getTimeout(config, commandKey, "ReadTimeout",
         IClientConfigKey.Keys.ReadTimeout, RibbonClientConfiguration.DEFAULT_READ_TIMEOUT);
      int ribbonConnectTimeout = getTimeout(config, commandKey, "ConnectTimeout",
         IClientConfigKey.Keys.ConnectTimeout, RibbonClientConfiguration.DEFAULT_CONNECT_TIMEOUT);
      int maxAutoRetries = getTimeout(config, commandKey, "MaxAutoRetries",
         IClientConfigKey.Keys.MaxAutoRetries, DefaultClientConfigImpl.DEFAULT_MAX_AUTO_RETRIES);
      int maxAutoRetriesNextServer = getTimeout(config, commandKey, "MaxAutoRetriesNextServer",
         IClientConfigKey.Keys.MaxAutoRetriesNextServer, DefaultClientConfigImpl.DEFAULT_MAX_AUTO_RETRIES_NEXT_SERVER);
     //在这里可以看到具体的计算方法
      ribbonTimeout = (ribbonReadTimeout + ribbonConnectTimeout) * (maxAutoRetries + 1) * (maxAutoRetriesNextServer + 1);
   }
   return ribbonTimeout;
}
```



ribbon的超时时间是ReadTimeout+connectTimeout一起计算的

hystrixTimeout要大于ribbonTimeout，否则hystrix熔断了以后，ribbon的重试就都没有意义了

比如上面的配置

zuul-url超时时间 =  (1000ms+300ms)*(0+1)*(1+1) = 2600ms 

 ribbbon超时时间 = (1000ms+300ms)*(0+1)*(1+1) = 2600ms 

 hystrix超时时间=15000ms

所以当请求超过2600ms时，处罚重试，当等到15000ms就会熔断

