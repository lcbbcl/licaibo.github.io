---
layout: post_layout
title: SpringCloud构建微服务（一） 配置中心
time: 2017年09月25日 星期一
location: 海南 东方
pulished: true
---

**SpringCloud提供了config的分布式配置中心，允许将我们的配置信息统一进行管理，每个服务启动时，根据配置文件名字以及环境动态的去加载配置。这里我先搭建一个配置中心，为后续服务提供者或是服务调用者提供动态配置。**

## 新建配置文件仓库
**SpringCloud Config允许我们存储在git或是svn，这里我选择的是git仓库进行存储。在我的GitHub新建config-repo仓库，用于存放service-dev.properties配置文件，里面我写的是Mysql的配置。注意：配置文件遵循SpringBoot提供的规范，这里xxx-dev指的是开发环境的配置**

```java
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/your_dadabase_name?useUnicode=true&characterEncoding=utf-8
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.initialSize=5
spring.datasource.minIdle=5
spring.datasource.maxActive=20
spring.datasource.maxWait=60000
spring.datasource.timeBetweenEvictionRunsMillis=3000
spring.datasource.minEvictableIdleTimeMillis=300000
spring.datasource.validation-query=select 1 from dual
spring.datasource.maxPoolPreparedStatementPerConnectionSize=20
spring.datasource.testOnBorrow=true
spring.datasource.testOnReturn=true

mybatis.mapper-locations=classpath:mapper/**/*.xml
mybatis.config-location=classpath:config/mybatis-config.xml
```

> *  **新建SpringBoot工程config-server（具体到仓库查看代码）**

```java
@Configuration
@EnableAutoConfiguration
@EnableDiscoveryClient
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class,args);
    }
}
```

> *  **新建application.yml配置文件，往eureka注册，配置git仓库的用户名和密码**

```java
server:
  port: 5555

spring:
  application:
    name: config-server

management:
  context-path: /admin

logging:
  level:
    com.netflix.discovery: 'OFF'
    org.springframework.cloud: 'INFO'

eureka:
  instance:
    leaseRenewalIntervalInSeconds: 10
    statusPageUrlPath: /admin/info
    healthCheckUrlPath: /admin/health
  client:
      serviceUrl:
        defaultZone: http://localhost:8765/eureka/

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/licaibo/config-repo.git #配置文件git仓库地址
          username: xxx
          password: xxx
```

> *  **启动eureka-server，接着再启动config-server。到eureka注册中心查看，会发现config服务的注册信息**

![cmd-markdown-logo](https://licaibo.github.io/assets/img/eureka-info.png)

> *  **访问http://localhost:5555/service-dev/properties，能够成功查看到仓库里service-dev.properties的配置信息。接下来，将使用其Mysql配置搭建服务的提供者和消费者**

![cmd-markdown-logo](https://licaibo.github.io/assets/img/config-info.png)