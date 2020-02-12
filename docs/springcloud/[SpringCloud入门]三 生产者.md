1.简介

模拟一个服务作为业务的提供者，为其他服务提供服务

2.pom

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
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
    

</dependencies>
```

3.application.yml

```
#服务名称
spring:
  application:
    name: producer-server
#服务启动端口
server:
  port: 10090
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
```

PS:服务健康检查是springboot提供的actuator监控器可以让注册中心和注册的服务不断进行心跳校验，检测服务是否活着

4.启动代码中增加注解

```
@EnableDiscoveryClient
```

声明服务可以被注册中心发现，从Spring Cloud Edgware开始可以不使用该注解，只要引用了spring-cloud-starter-netflix-eureka-server就可以被注册中心发现

5.DEMO

```java
package com.neo.producer.controller;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * <p></p>
 *
 * @author neo
 * @date 2020/1/19 13:16
 */
@Slf4j
@RestController
public class DemoController {
    @Value("${server.port}")
    private String port;

    @RequestMapping("/hello")
    public String hello(@RequestParam String name) {
        log.info("[生产者服务] 接口测试 name:{}", name);
        return String.format("Hello %s ,服务[%s]被调用", name, port);
    }
}

```

