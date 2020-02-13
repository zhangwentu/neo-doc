1.简介

Eureka是Netflix开源的一款提供服务注册和发现的产品，它提供了完整的Service Registry和Service Discovery实现。也是springcloud体系中最重要最核心的组件之一。

主要功能：注册、发现、熔断、负载、降级等

相当于微服务的中枢，协调各个器官功能。

以前的服务可能互相之间的调用是各种交织错乱的 如下图：

<p align="center">
    <a href="https://tva1.sinaimg.cn/large/0082zybpgy1gbtl7xs3vsj315c0u0q5c.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbtl7xs3vsj315c0u0q5c.jpg" width=""/>
    </a>
</p>

当使用注册中心之后可以有效的将服务管理起来所有的服务只向注册中心暴露自己的请求和服务

服务间的关系将会如下图：

<p align="center">
    <a href="https://tva1.sinaimg.cn/large/0082zybpgy1gbtllqiy89j31gm0qeq54.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbtllqiy89j31gm0qeq54.jpg" width=""/>
    </a>
</p>





2.pom

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>

</dependencies>
```

3.application.yml

```
#服务名称
spring:
  application:
    name: eureka-server
#服务启动端口
server:
  port: 10086
logging:
  config: classpath:logback.xml


#主机地址
eureka:
  instance:
    hostname: localhost
  #这两个false表明这个服务为eureka server
  client:
    fetch-registry: false
    register-with-eureka: false
    #服务地址
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

4.启动代码中增加注解

```
@EnableEurekaServer
```

5.启动服务后访问地址

http://127.0.0.1:10086/

6.集群注册中心

新建另一个注册中心 分别将两个注册中心的服务地址指向对方

<p align="center">
    <a href="https://tva1.sinaimg.cn/large/0082zybpgy1gbtmg1gikcj317s0n0q69.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbtmg1gikcj317s0n0q69.jpg" width=""/>
    </a>
</p>

<p align="center">
    <a href="https://tva1.sinaimg.cn/large/0082zybpgy1gbtmib5jsnj31580nedj5.jpg" target="_blank">
        <img src="https://tva1.sinaimg.cn/large/0082zybpgy1gbtmib5jsnj31580nedj5.jpg" width=""/>
    </a>
</p>

再将应用的注册中心同时指定这两个注册中心，

```
eureka.client.service-url.defaultZone:  http://127.0.0.1:10086/eureka/,http://127.0.0.1:10088/eureka/
```

这样只要任何一个注册中心掉线 另外一个都可以独立承担注册中心的责任，当恢复后又可以从存活的注册中心复制所有的服务