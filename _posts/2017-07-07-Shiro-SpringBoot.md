---
layout: post_layout
title: SpringBoot使用Shiro进行权限认证(一)
time: 2017年07月07日 星期五
location: 海南 海口
pulished: true
---

在日常项目开发中，权限认证是不可少的模块。比较常用的有Spring Security，或是轻量级的Apache Shiro。相对来说Shiro提供了认证、授权、加密、会话管理、与Web集成、缓存等。这些都是日常会用到的，而且Shiro的API比较简洁，学习成本相对低。接下来将整理一下在SpringBoot中如何集成Shiro：

> * RBAC(Role-Based Access Control )基于角色访问控制
> * SpringBoot集成Shiro和配置
> * Shiro的登录和认证
> * 当前不足点

## (一) RBAC介绍
RBAC基于角色访问控制，在权限设计上用户是基于角色进行权限认证，而角色又是和资源相关联的。这样在设计和管理上简化了权限的操作，它们都是层级依赖，更方便我们的管理。如此一来，数据库表设计可以如下图：
![cmd-markdown-logo](https://licaibo.github.io/assets/img/RBAC.png)

##(二) SpringBoot集成Shiro和配置
#### 1、导入Shiro依赖
在pom.xml中加入以下依赖

```java
<dependency>
        <groupId>org.apache.shiro</groupId>
        <artifactId>shiro-spring</artifactId>
        <version>1.2.5</version>
</dependency>
```
### 2、配置Shiro
在配置之前，首先了解一下Shiro中主要功能，并看看它们主要是做什么的。

> * **Subject** 安全视角下与软件交互的实体（用户，第三方服务等等）
> * **Authenticator** 用户登录时进行账户的认证
> * **Authorizer** 用户登录时进行账户的权限资源认证
> * **Realms** 每当执行认证或授权时，shiro会从程序配置的一个或多个Realm中查询

新建JAVA类ShiroRealm用于继承Shiro的AuthorizingRealm抽象类，并复写doGetAuthenticationInfo和doGetAuthorizationInfo用于账户和权限的认证。

```java
public class ShiroRealm extends AuthorizingRealm {

    @Autowired
    private UserService userService;

    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        UsernamePasswordToken token = (UsernamePasswordToken) authenticationToken;
        String userName = token.getUsername();
        String password = new String((char[])token.getCredentials());
        //这里可以自行查询数据库进行验证
        if(userName != "zhangshan" && password != DigestUtils.md5Hex("123456")) {
            throw new UnknownAccountException ("未知的账户认证失败");
        }
        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(
                userName, //用户名
                DigestUtils.md5Hex("123456"), //密码
                getName()  //realm name
        );
        //存入session
        //SecurityUtils.getSubject().getSession().setAttribute("user", user);
        return authenticationInfo;
    }

    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        String useCode = (String) principalCollection.getPrimaryPrincipal();
        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
        Map<String,Object> userAuthorityInfoMap = userService.getUserAuthorityInfo(useCode);
        //这里可以从数据库进行查询
        List<String> roles = new ArrayList<String>();
        List<String> perms = mew ArrayList<String>();
        roles.add("管理员");
        perms.add("查看用户模块");
        simpleAuthorizationInfo.addRoles(roles);
        simpleAuthorizationInfo.addStringPermissions(perms);
        return simpleAuthorizationInfo;
    }

}
```

在SpringBoot中配置Shiro，配置URL过滤规则

```java
@Configuration
public class ShiroConfiguration {

    /**
     * 负责org.apache.shiro.util.Initializable类型bean的生命周期的，初始化和销毁。
     */
    @Bean(name = "lifecycleBeanPostProcessor")
    public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }

    /**
     * shiro密码认证配置,使用MD5 HEX16进制
     */
    @Bean(name = "hashedCredentialsMatcher")
    public HashedCredentialsMatcher hashedCredentialsMatcher() {
        HashedCredentialsMatcher credentialsMatcher = new HashedCredentialsMatcher();
        credentialsMatcher.setHashAlgorithmName("MD5");
        credentialsMatcher.setHashIterations(1);//散列的次数,相当于md5("");
        credentialsMatcher.setStoredCredentialsHexEncoded(true);//采用HEX16进制编码
        return credentialsMatcher;
    }

    /**
     * ShiroRealm，需要自己实现自定义的认证类，继承自AuthorizingRealm，负责用户的认证和权限的处理
     */
    @Bean(name = "shiroRealm")
    @DependsOn("lifecycleBeanPostProcessor")
    public ShiroRealm shiroRealm() {
        ShiroRealm realm = new ShiroRealm();
        //realm.setCacheManager(ehCacheManager());
        return realm;
    }

    /**
     * EhCacheManager，缓存管理，用户登陆成功后，把用户信息和权限信息缓存起来，
     * 然后每次用户请求时，放入用户的session中，如果不设置这个bean，每个请求都会查询一次数据库。
     */
//    @Bean(name = "ehCacheManager")
//    @DependsOn("lifecycleBeanPostProcessor")
//    public EhCacheManager ehCacheManager() {
//        EhCacheManager ehcacheManager = new EhCacheManager();
//        //ehcacheManager.setCacheManagerConfigFile("classpath:config/ehcache-shiro.xml");
//        return ehcacheManager;
//    }

    /**
     * SecurityManager权限管理，主营管理登陆，登出，权限，session的处理
     */
    @Bean(name = "securityManager")
    public DefaultWebSecurityManager securityManager() {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        securityManager.setRealm(shiroRealm());
        //securityManager.setCacheManager(ehCacheManager());
        return securityManager;
    }

    /**
     * ShiroFilterFactoryBean配置URL过滤链规则
     */
    @Bean(name = "shiroFilter")
    public ShiroFilterFactoryBean shiroFilterFactoryBean() {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        shiroFilterFactoryBean.setLoginUrl("/login");
        shiroFilterFactoryBean.setSuccessUrl("/index");
        shiroFilterFactoryBean.setUnauthorizedUrl("/pages/403");
        Map<String, String> filterChainDefinitionMap = new LinkedHashMap<String, String>();
        //Shiro拦截URL规则
        filterChainDefinitionMap.put("/logout", "logout");
        filterChainDefinitionMap.put("/js/**", "anon");
        filterChainDefinitionMap.put("/css/**", "anon");
        filterChainDefinitionMap.put("/login", "anon");
        filterChainDefinitionMap.put("/403", "anon");
        filterChainDefinitionMap.put("/userInfo/**", "authc,perms[查看用户模块]");
        filterChainDefinitionMap.put("/messageInfo/**", "authc,roles[管理员]");
        filterChainDefinitionMap.put("/**", "authc");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;
    }

    /**
     * 由Advisor决定对哪些类的方法进行AOP代理。
     */
    @Bean
    @ConditionalOnMissingBean
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator defaultAAP = new DefaultAdvisorAutoProxyCreator();
        defaultAAP.setProxyTargetClass(true);
        return defaultAAP;
    }

    /**
     * shiro里实现的Advisor类，内部使用AopAllianceAnnotationsAuthorizingMethodInterceptor来拦截用以下注解的方法。
     */
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor() {
        AuthorizationAttributeSourceAdvisor aASA = new AuthorizationAttributeSourceAdvisor();
        aASA.setSecurityManager(securityManager());
        return aASA;
    }

}
```

## (三) Shiro的登录和认证
新建LoginController进行登录验证Shiro。在subject.login(token)这行代码执行的时候，Shiro会回调我们上面实现的ShiroRealm进行账户认证和权限认证。

```java
@RequestMapping("/login")
@Controller
public class LoginController {

    private static Logger logger = LoggerFactory.getLogger(LoginController.class);

    @RequestMapping(method = RequestMethod.GET)
    @ResponseBody
    public String loginPage(){
        return "login";
    }

    @RequestMapping(value = "/check",method = RequestMethod.POST)
    @ResponseBody
    public String login(@RequestParam("userName") String userName, @RequestParam("passWord") String passWord){
        Subject subject = SecurityUtils.getSubject();
        UsernamePasswordToken token = new UsernamePasswordToken(userCode, DigestUtils.md5Hex(passWord));
        try {
            subject.login(token);
        } catch (Exception e) {
            logger.error("用户用户名和密码..验证未通过");
            return "redirect:/login";
        }
        // catch (UnknownAccountException e) {
            // logger.error("未知的账户认证失败);
        // } catch (DisabledAccountException e) {
            // logger.error("账户已经被禁用,认证失败");
        // }

         return "/index";
    }

}
```

## (四) 当前不足点
到这里Shiro已经可以进行登录验证，在配置中也可以对指定的URL拦截，并通过角色和资源认证才可以访问。但是细心想想，还是有一些不足，在一下节将对其进行优化，主要有以下几点
> * 不支持AJAX的调用，目前只能重定向页面，这对于一些前后端分离的开发是不满足的
> * 不支持前台FreeMarker模板页面元素的权限控制，包括按钮、文字等
> * URL拦截冗余在代码，虽然说SpringBoot提倡JAVA Configuration配置，但我们还是想要单独写到配置文件，例如yml或是properties