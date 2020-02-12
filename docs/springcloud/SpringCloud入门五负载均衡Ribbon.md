1.简介

Ribbon是一个基于HTTP和TCP的客户端负载均衡工具,它几乎存在于每一个Spring Cloud构建的微服务和基础设施中,不需要额外的部署，比如常用的feign就用到了ribbon, 以及后面会讲到的网关(zuul)都会用到ribbon.

下面将演示 服务消费者直接通过调用被@LoadBalanced注解修饰过的RestTemplate来实现面向服务的接口调用。

2.pom

在消费者服务里追加

```
<!-- ribbon -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

3.启动代码中增加RestTemplate声明

```java
package com.neo.customer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

/**
 * <p></p>
 *
 * @author neo
 * @date 2020/1/16 15:32
 */
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class CustomerServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(CustomerServerApplication.class, args);
    }

    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

4.增加RibbonRestService.java

```java
package com.neo.customer.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

/**
 * <p></p>
 *
 * @author neo
 * @date 2020/2/12 15:58
 */
@Service
public class RibbonRestService {
    @Autowired
    RestTemplate restTemplate;

    public String test(String name) {
        //这里因为注册到了注册中心，域名可以直接使用对应服务的 application name
        return restTemplate.getForObject("http://producer-server/hello?name=" + name, String.class);
    }
}
```

5.DEMO

测试代码

```java
@Test
public void ribbonTest() {
    log.info(ribbonRestService.test("1"));
    log.info(ribbonRestService.test("2"));
    log.info(ribbonRestService.test("3"));
    log.info(ribbonRestService.test("4"));
    log.info(ribbonRestService.test("5"));
    log.info(ribbonRestService.test("6"));

}
```

实现效果：

<p align="center">
    <a href="https://tva1.sinaimg.cn/large/0082zybpgy1gbtolqn24bj31ju05qn0j.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbtolqn24bj31ju05qn0j.jpg" width=""/>
    </a>
</p>

