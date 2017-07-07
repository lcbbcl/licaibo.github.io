---
layout: post_layout
title: SpringBoot使用Shiro进行权限认证(二)
time: 2017年07月07日 星期五
location: 海南 海口
pulished: true
---

在上一节中实现了在SpringBoot中使用Shiro做权限控制，但是针对上一节留下的不足点，在这里进行一下优化和改造，主要有一下几点:
> * **支持AJAX请求**
> * **支持FreeMarker模板**
> * **URL拦截提取到yml配置文件**

## (一) 支持AJAX请求
**如果是AJAX请求URL接口，没有登录或是没有权限的时候，我们希望是返回指定的JSON格式，而不是进行页面的重定向。当时这个问题也是困扰很久，好在Shiro在这方面提供了很好的扩展，在google帮助下，我决定使用的方案是重写Shiro提供的Filter过滤器，在没有登录或是角色权限认证失败时进行处理。如果是AJAX请求返回指定JSON，普通请求进行页面的重定向。于是乎我找到如下Shiro Filter的关系图：**

![cmd-markdown-logo](http://img.blog.csdn.net/20170601195007717?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxNDA0MjE0Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> * **AdviceFilter 过滤器类似于类似于SpringMVC中的HandlerInterceptor拦截器，用于在请求URL之前进行拦截，主要处理一些是否登录的验证**
> * **PermissionsAuthorizationFilter提供了onAccessDenied方法，主要用于perms资源认证失败时回调onAccessDenied进行一些处理**
> * **RolesAuthorizationFilter提供了onAccessDenied方法，主要用于roles角色认证失败时回调onAccessDenied进行一些处理**

### (1) 重写AdviceFilter过滤器

```java
class ShiroAdviceFilter extends AdviceFilter {

    /**
     * 在访问URL前判断是否登录
     * @param request
     * @param response
     * @return true-继续往下执行，false-该filter过滤器已经处理，不继续执行其他过滤器
     * @throws Exception
     */
    @Override
    protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        User user = (User) httpServletRequest.getSession().getAttribute("user");
        if (null == user && !StringUtils.contains(httpServletRequest.getRequestURI(), "/login")
                && !StringUtils.contains(httpServletRequest.getRequestURI(), "/js")
                && !StringUtils.contains(httpServletRequest.getRequestURI(), "/images")) {
            String requestedWith = httpServletRequest.getHeader("X-Requested-With");
            if (StringUtils.isNotEmpty(requestedWith) && StringUtils.equals(requestedWith, "XMLHttpRequest")) {//如果是ajax返回指定数据
                JSONObject json = new JSONObject();
                json.put("statuscode":500);
                json.put("message":"没有登录");
                httpServletResponse.setCharacterEncoding("UTF-8");
                httpServletResponse.setContentType("application/json");
                httpServletResponse.getWriter().write(JSONObject.toJSONString(json));
                return false;
            } else {//不是ajax进行重定向处理
                httpServletResponse.sendRedirect("/login");
                return false;
            }
        }
        return true;
    }

}
```
### (2) Filter过滤器工具类
```java
public class FilterUtils {

    /**
     * 如果是ajax返回指定格式数据，如果是普通请求重定向页面
     * @param servletRequest
     * @param servletResponse
     * @throws IOException
     */
    public static void AuthorizationOnAccessDenied(ServletRequest servletRequest, ServletResponse servletResponse) throws IOException {
        HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
        HttpServletResponse httpServletResponse = (HttpServletResponse) servletResponse;
        String requestedWith = httpServletRequest.getHeader("X-Requested-With");
        if (StringUtils.isNotEmpty(requestedWith) &&
                StringUtils.equals(requestedWith, "XMLHttpRequest")) {//如果是ajax返回指定格式数据
            JSONObject json = new JSONObject();
            json.put("statuscode":403);
            json.put("message":"角色资源认证失败");
            httpServletResponse.setCharacterEncoding("UTF-8");
            httpServletResponse.setContentType("application/json");
            httpServletResponse.getWriter().write(JSONObject.toJSONString(responseHeader));
        } else {//如果是普通请求进行重定向
            httpServletResponse.sendRedirect("/403");
        }
    }

}
```

### (3) 重写PermissionsAuthorizationFilter
```java
public class ShiroPermsFilter extends PermissionsAuthorizationFilter {

    /**
     * shiro认证perms资源失败后回调方法
     * @param servletRequest
     * @param servletResponse
     * @return
     * @throws IOException
     */
    @Override
    protected boolean onAccessDenied(ServletRequest servletRequest, ServletResponse servletResponse) throws IOException {
        FilterUtils.AuthorizationOnAccessDenied(servletRequest,servletResponse);
        return false;
    }
}
```

### (4) 重写RolesAuthorizationFilter过滤器
```java
public class ShiroRolesFilter extends RolesAuthorizationFilter {
    /**
     * shiro认证roles角色失败后回调方法
     * @param servletRequest
     * @param servletResponse
     * @return
     * @throws IOException
     */
    @Override
    protected boolean onAccessDenied(ServletRequest servletRequest, ServletResponse servletResponse) throws IOException {
        FilterUtils.AuthorizationOnAccessDenied(servletRequest,servletResponse);
        return false;
    }
}
```

### (5) 在上一节Shiro配置类ShiroConfiguration对我们重写的Filter过滤器进行配置，修改如下
```java

    //增加ShiroLoginFilter实例化
    @Bean(name = "shiroLoginFilter")
    public ShiroLoginFilter shiroLoginFilter(){
        ShiroLoginFilter shiroLoginFilter = new ShiroLoginFilter();
        return shiroLoginFilter;
    }

    //修改shiroFilterFactoryBean配置
    @Bean(name = "shiroFilter")
    public ShiroFilterFactoryBean shiroFilterFactoryBean() {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager());
        Map<String, Filter> filtersMap = new LinkedHashMap<String, Filter>();
        filtersMap.put("shiroLoginFilter", shiroLoginFilter());
        filtersMap.put("perms", new ShiroPermsFilter());
        filtersMap.put("roles",new ShiroRolesFilter());
        shiroFilterFactoryBean.setFilters(filtersMap);
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
```
**可以看到我们用的是LinkedHashMap将我们写的ShiroLoginFilter、ShiroPermsFilter、ShiroRolesFilter存储起来并设置。这代表用户每一次URL请求，都会按顺序的经过ShiroLoginFilter -> ShiroPermsFilter -> ShiroRolesFilter处理，只有全部都通过认证才会请求成功，否则将返回JSON或是重定向。这样Shiro既能兼容普通请求的重定向，也能在AJAX时返回JSON，流程图如下：**
![cmd-markdown-logo](https://licaibo.github.io/assets/img/Shiro.png)

## (二) 支持FreeMarker模板
**Shiro本身是通提供了JSP的一套标签库，用于在JSP进行权限的控制，对FreeMarker官方是没有提供。但是有大神将其开源到了GitHub，参考[shiro-freemarker-tags](https://github.com/jagregory/shiro-freemarker-tags)。下面讲述的是，在SpringBoot项目中使用shiro-freemarker-tags**
### (1) 在pom.xml中导入依赖
```java
<dependency>
        <groupId>net.mingsoft</groupId>
        <artifactId>shiro-freemarker-tags</artifactId>
        <version>0.1</version>
</dependency>
```
### (2) 新建FreeMarkerConfig类用于配置FreeMarkers标签
```java
@Service
public class FreeMarkerConfig implements InitializingBean {

       @Autowired
       private Configuration configuration;

       @Override
       public void afterPropertiesSet() throws Exception {
           configuration.setSharedVariable("shiro", new ShiroTags());
       }

}
```
### (3) 在FreeMarker页面使用标签库进行权限控制
```xml
<@shiro.hasRole name="ADMIN">
    <p>只有ADMIN的角色才能看到这段文字</p>
</@shiro.hasRole>

<@shiro.hasAnyRoles name="admin,user,member">
    <p>只有admin或user或member其中一个角色才能看到这段文字</p>
</@shiro.hasAnyRoles>

<@shiro.lacksRole name="admin">
    <p>只有不拥有admin角色才能看到这段文字</p>
</@shiro.lacksRole>

<@shiro.hasPermission name="查看用户模块">
	 <p>只有拥有【查看用户模块】资源的用户才能看到这段文字</p>
</@shiro.hasPermission>

<@shiro.guest>
    您当前是游客，关于更多使用请自行查阅标签库
</@shiro.guest>
```

## (三) URL拦截提取到yml配置文件
**之前在ShiroConfiguration配置类的shiroFilterFactoryBean方法中，我们拦截的URL使用LinkedHashMap写在了代码里面，这样后期维护和可观性不够好，我们可以把URL的拦截单独提取到yml配置文件。这里不选择properties文件，是因为properties文件是不支持有序的，这里URL的顺序请务必保持有序的加载。**
### (1) 在yml文件中配置我们的URL拦截规则
```xml
shiro:
    filterUrl:
      /logout: logout
      /js/**: anon
      /css/**: anon
      /images/**: anon
      /login/**: anon
      /logout/**: anon
      /403/**: anon
      /userInfo/**: authc,perms[查看用户模块]
      /messageInfo/**: authc,roles[管理员]
      /**: authc
```

### (2) 新建ShiroUrlStrategy用于从yml加载配置
```java
@EnableConfigurationProperties
@ConfigurationProperties(prefix = "shiro") //yml配置文件的前缀
public class ShiroUrlStrategy {

    private LinkedHashMap<String, String> filterChainDefinitionMap;

    public LinkedHashMap<String, String> getFilterChainDefinitionMap() {
        return filterChainDefinitionMap;
    }

    public void setFilterChainDefinitionMap(LinkedHashMap<String, String> filterChainDefinitionMap) {
        this.filterChainDefinitionMap = filterChainDefinitionMap;
    }
}
```

### (3) 将ShiroConfiguration配置类的shiroFilterFactoryBean方法
```java
//修改shiroFilterFactoryBean配置
    @Bean(name = "shiroFilter")
    public ShiroFilterFactoryBean shiroFilterFactoryBean() {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager());
        Map<String, Filter> filtersMap = new LinkedHashMap<String, Filter>();
        filtersMap.put("shiroLoginFilter", shiroLoginFilter());
        filtersMap.put("perms", new ShiroPermsFilter());
        filtersMap.put("roles",new ShiroRolesFilter());
        shiroFilterFactoryBean.setFilters(filtersMap);
        //从配置获取URL拦截规则
        ShiroUrlStrategy shiroUrlStrategy = ShiroUrlStrategy();
        LinkedHashMap<String, String> filterChainDefinitionManager = shiroUrlFiler.getFilterChainDefinitionMap();
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionManager);
        return shiroFilterFactoryBean;
    }
```
