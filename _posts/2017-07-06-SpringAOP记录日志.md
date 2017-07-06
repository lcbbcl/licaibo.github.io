---
layout: post_layout
title: Spring AOP 自定义注解记录操作日志
time: 2017年07月06日 星期四
location: 海南
pulished: true
---


# Spring AOP 自定义注解记录操作日志

------

在项目开发中，通常我们会记录一些用户操作上的日志，主要有修改人、修改时间、修改内容等等，以便于后续的问题排查和分析。最近在开发时，刚好需要在用户操作时，记录相关日志。在参考了网上的方案后，决定使用自定义注解和AOP的方法。面向切面的编程，就算是记录日志出错了也不影响到主流程业务。


> * 自定义注解@Log用于标识需要记录日志的方法
> * 定义一个AOP切面对方法进行拦截
> * 将拦截的信息记录到日志表




------

## 1、自定义注解

自定义注解用于标识需要拦截的方法，可以是主题和类型等，具体视情况而定

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface Log {

    /** * 主题(新增或是修改) * @return */
    String item() default "";

    /** * 类型(用户信息修改等) * @return */
    String type() default "";

}
```

## 2、定义Spring AOP切面
Spring AOP提供了几种注解用于不同的场景。我的工程是SpringBoot项目，在这里选择了@AfterReturning，在方法执行完后再去执行，并且用@Async声明了该方法是异步的执行。
注：SpringBoot使用@Async需要在启动时加上@EnableAsync
> * @Before 在拦截方法执行前执行
> * @After 在拦截方法执行之后执行。
> * @AfterReturning 在拦截方法返回后执行
> * @AfterThrowing 在拦截方法抛出异常后执行
> * @Around 是可以同时在所拦截方法的前后执行

```java

@Aspect
@Component
public class LogAop {

private static final Logger logger=LoggerFactory.getLogger(FileOperateLogAop.class);

    //声明AOP切入点，凡是使用了@Log的方法均被拦截
    @Pointcut("@annotation(com.lcb.demo.annotation.Log)")
    public void logPointcut() { }

    @Async
    @AfterReturning(pointcut = "logPointcut() && @annotation(log)",returning = "returnValue")
    public void afterFileOperate(JoinPoint joinPoint,
                                 Log log,
                                 Object returnValue) {
        logger.info("开始记录【{}】操作的日志信息",chargeIten);
        
        //获取注解的属性值
        String iten = log.item();
        String type = log.type();
        //获取拦截方法的参数
        Object[] objects = joinPoint.getArgs();

        //TODO 实现自己对日志记录的逻辑
        
        logger.info("结束记录【{}】操作的日志信息",chargeIten);

    }

}

```

## 3、声明@Log注解

```java

@Log(item = "用户信息",type="修改")
public String testLog(String text) {
        return "abc";
    }

```

