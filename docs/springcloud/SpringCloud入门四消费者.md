

1.简介

在springcloud中所有的服务都注册到了注册中心，所以在服务间调用的时候可以使用feign，通过指定服务的服务名，不再需要配置对应的iP和域名，这些工作可以交给注册中心来做了。

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
    <!-- spring boot Test依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- 服务注册 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <!-- 监控神器 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!-- feign -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>


</dependencies>
```

3.application.yml

```
#服务名称
spring:
  application:
    name: customer-server
#服务启动端口
server:
  port: 10091
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

4.启动代码中增加注解

```
@EnableDiscoveryClient
@EnableFeignClients
```

EnableFeignClients用来开启feign

5.feign调用接口

```
package com.neo.customer.service;

import com.neo.customer.service.hystrix.ProducerServiceHystric;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;

/**
 * <p></p>
 *
 * @author neo
 * @date 2020/2/11 17:11
 */
@FeignClient(name = "producer-server")
public interface IFeignService {
    @RequestMapping("/hello")
    String hello(@RequestParam(name = "name") String name);
}
```

6.DEMO

做两个demo一个controller访问 一个是test 单元测试

```java
package com.neo.customer.controller;

import com.neo.customer.service.IFeignService;
import org.springframework.beans.factory.annotation.Autowired;
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
public class FeignServiceController {
    @Autowired
    private IFeignService iFeignService;

    @RequestMapping(value = "/test")
    public String test(@RequestParam String name) {

        return iFeignService.hello(name);
    }
}
```

```java
package com.neo.customer;

import com.neo.customer.service.IFeignService;
import lombok.extern.slf4j.Slf4j;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * <p></p>
 *
 * @author neo
 * @date 2020/2/11 17:14
 */
@Slf4j
@RunWith(SpringRunner.class)
@SpringBootTest(classes = CustomerServerApplication.class)
public class FeignServiceTest {

    @Autowired
    private IFeignService iFeignService;

    @Test
    public void handle() {
        String res = iFeignService.hello("test");
        log.info(res);
    }

}
```