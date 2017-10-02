---
layout: post_layout
title: SpringCloud构建微服务（三） 服务消费
time: 2017年10月02日 星期一
location: 海南 东方
pulished: true
---

**本节基于前面搭建的eureka注册中心和config配置中心，要搭建一个服务提供者并集成mybatis用于数据查询，mysql配置从配置中心进行加载，搭建一个服务消费者对提供者进行调用。配置的加载如下图**

![cmd-markdown-logo](https://licaibo.github.io/assets/img/config-server.png)

## (一) 服务提供者
> * **新建名为provider-server的SpringBoot工程，端口8081，加载配置中心的service-dev.properties，往euraka进行注册，配置bootstrap.yml如下**

```java
server:
  port: 8081

spring:
  cloud:
    config:
      uri: http://localhost:${config.port:5555}
      name: service
      profile: ${config.profile:dev}
  application:
    name: provider-server

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8765/eureka/

pagehelper:
  rowBoundsWithCount: true
  pageSizeZero: true
  reasonable: false
```

> * **在pom.xml中引入mybatis依赖**

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.2.0</version>
</dependency>
```
> * 写一个简单的查询和接口，代码如下：

```java
@Mapper
public interface UserDao {

    @Select("SELECT * FROM user WHERE NAME = #{name}")
    User selectByName(@Param("name") String name);

}
```

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/{name}")
    public User selectByName(@PathVariable String name) {
        return userService.selectByName(name);
    }

}
```

> * 启动eureka-server和config-server工程，再接着启动provider-server，然后访问http://localhost:8081/user/lemo，返回user表里的数据如下。工程里面也集成了swagger，一个简单方便的接口调试工具，地址为http://localhost:8081/swagger-ui.html

```java
{"id":1,"name":"lemo","age":21,"address":"海南省海口市"}
```

## (二) 服务消费者
> * **新建名为consumer-server的SpringBoot工程，端口8082，用于调用服务的提供者provider-server，配置bootstrap.yml如下**

```java
server:
  port: 8082

spring:
  application:
    name: consumer-server

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8765/eureka/
```

> * **调用的provider-server查询接口代码如下。这里注意只要写出服务名即可，不需要具体ip。因为它们都在eureka注册中心进行注册和一治理，eureka根据服务名即可找到相应的服务**

```java
@Service
public class UserServicer {

    @Autowired
    RestTemplate restTemplate;

    public ResponseEntity<User> selectByName(String name) {
        ResponseEntity<User> responseEntity = restTemplate.getForEntity("http://provider-server/user/{name}" ,User.class,name);
        return responseEntity;
    }

}
```

```java
@RestController
@RequestMapping("user")
public class UserController {

    @Autowired
    private UserServicer userServicer;

    @GetMapping("/{name}")
    public User selectByName(@PathVariable String name) {
       return userServicer.selectByName(name).getBody();
    }

}
```

> * **启动eureka-server和config-server工程，再接着启动provider-server，最后启动consumer-server，访问http://localhost:8082/user/lemo，返回服务提供者查询的数据**

```java
{"id":1,"name":"lemo","age":21,"address":"海南省海口市"}
```

> * **访问http://localhost:8765查看eureka面板，发现此时注册的服务有config-server、provider-server、consumer-server**

![cmd-markdown-logo](https://licaibo.github.io/assets/img/eureka-server.png)


