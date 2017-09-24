---
layout: post_layout
title: SpringCloud构建微服务（一） Eureka
time: 2017年09月24日 星期日
location: 海南 东方
pulished: true
---

**自己用SpringBoot和SpringCloud也有一年多的时间了，第一次接触到SpringBoot就被它的快速简单搭建给吸引，加上公司之前有做SpringCloud的微服务开发，这里想分享一下SpringCloud搭建简单的微服务，包括搭建注册中心、配置中心，服务的负载均衡调用等等，工程全部都是使用SpringBoot进行搭建，算是一个学习总结吧，代码都会托管在自己GitHub的SpringCloud-Project仓库**

## 注册中心Eureka搭建

**当我们的系统都被拆分成一个个细小的服务时，对于服务的统一治理则是我们构建微服务架构的首选，所有的服务都围绕着eureka进行注册、下线，大致上可以如下图所示：**
![cmd-markdown-logo](https://licaibo.github.io/assets/img/eureka.png)



> *  **新建Maven工程eureka-server，pom.xml如下（具体到仓库查看代码）**

```java
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.3.5.RELEASE</version>
</parent>
    <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Brixton.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```

> *  **其实eureka也是一个SpringBoot工程。我们新建EurekaServerApplication.java，其中@EnableEurekaServer将其标示为eureka注册中心**

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class,args);
    }
}
```
> *  **增加配置文件application.yml**
```java
server:
  port: 8765

spring:
  application:
    name: eureka-serve

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:${server.port}/eureka/
    registerWithEureka: false  //不能将自己注册自己
    fetchRegistry: false    //由于自己是注册中心，所以不需要获取注册信息
  server:
    waitTimeInMsWhenSyncEmpty: 0  //设置同步为空时的等待时间
```

> *  **启动工程，并访问http://localhost:8765/，eureka面板如下图。可以看到现在eureka注册中心还没有服务进行注册，接下来将以eureka为基础，继续搭建我们的配置中心和服务提供者和消费者，都需要往eureka进行注册**

![cmd-markdown-logo](https://licaibo.github.io/assets/img/eureka-center.png)