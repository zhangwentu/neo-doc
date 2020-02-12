SpringCloud 入门 以实践为主体验

搭建版本：

```
<java.version>1.8</java.version>
<spring.boot.version>2.1.4.RELEASE</spring.boot.version>
<spring.cloud.version>Greenwich.RELEASE</spring.cloud.version>
```

1.[目录](./springcloud/[SpringCloud 入门]一 目录.md) 

2.[注册中心，eureka（单点+集群)](./springcloud/[SpringCloud 入门]二 注册中心.md) 

3.[服务生产者](./springcloud/[SpringCloud 入门]三 服务生产者.md) 

4.[服务消费者（feign）](./springcloud/[SpringCloud 入门]四 服务消费者.md) 

5.[负载均衡（ribbon）](./springcloud/[SpringCloud 入门]五 负载均衡Ribbon.md) 

6.[熔断器（hystrix）](./springcloud/[SpringCloud 入门]六 熔断器Hystrix.md) 

7.网关路由（zuul）

8.配置中心(apollo)

9.链路监控（slueth）

10.分布式事务（seata）



搭建项目：

1.先新建一个parent项目

   在父pom里统一管理依赖版本 在这里使用spring-cloud-dependencies依赖组

2.pom如下

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.t</groupId>
    <artifactId>neo-parent</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>
    <modules>
        <module>neo-cloud-eureka</module>
        <module>neo-cloud-producer</module>
        <module>neo-cloud-customer</module>
    </modules>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <spring.boot.version>2.1.4.RELEASE</spring.boot.version>
        <spring.cloud.version>Greenwich.RELEASE</spring.cloud.version>
 
    </properties>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring.boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring.cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
           
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.1</version>
                    <configuration>
                        <source>${maven.compiler.source}</source>
                        <target>${maven.compiler.target}</target>
                    </configuration>
                </plugin>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring.boot.version}</version>
                    <executions>
                        <execution>
                            <goals>
                                <goal>repackage</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
    <repositories>
        <repository>
            <id>alimaven</id>
            <name>aliyun maven</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

</project>
```