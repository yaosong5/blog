---
title: spring集成权限校验
date: 2018-05-08 18:34:55
tags: [技术,开发,Java]
categories: [Spring]
toc: true
---
# shiro简介  
shiro是权限控制的一个框架		
是一个强大易用的Java安全框架，提供了认证、授权、加密和会话管理功能，可为任何应用提供安全保障 - 从命令行应用、移动应用到大型网络及企业应用。
<!-- more -->

### **权限控制的方式**   
权限有四种实现方式   
注解(基于代理),url拦截(基于过滤器),shiro标签库(基于标签),编写代码(及其不推荐)   
**不论哪种方式:都需要引入spring用于整合shiro的过滤器  **   
web.xml中:DelegatingFilterProxy=>spring整合shiro
配置spring提供的用于整合shiro框架的过滤器
```xml
  <filter>
    <filter-name>shiroFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy
   </fileter>
```  
filet-name需要和**spring配置文件**中的一个BEAN对象的id保持一致**非常重要**  

###  配置   
I. 注解方式,注解是利用生成的代理对象来完成权限校验:   
spring框架会为当前action对象(加注解的action)创建一个代理对象,如果有权限,就执行这个方法,不然就会报**异常**
(将spring,Strust配置文件丰富:添加权限的注解,struts添加捕获异常,跳转页面)  
 1. 需要在spring配置文件中进行配置开启注解**DefaultAdvisorAutoProxyCreator**,
  并配置成cjlib方式的注解   
```xml
<property name="proxyTargetClass" value="true">\</property>
```
注解实现权限当为jdk模式的时候    
方法注解实现权限过滤  
抛异常的原因:因为如果是jdk方式的话,实现的接口modelDriven只有一个getModel方法      
所以不能进行对除该方法外其他方法进行注解

 2.  定义切面类**AuthorizationAttributeSourceAdvisor**
 ```xml
 <bean id="authorizationAttributeSourceAdvisor" class="org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor"></bean>
 ```

 3. 在需要权限才能访问的方法上添加注解
   ```java
  @RequiresPermissions("relo_delete这是权限名称")   
  ```  

II.  url拦截(springxml)
  基于过滤器或者拦截器实现   
```xml
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
		<property name="securityManager" ref="securityManager"/>
		<property name="loginUrl" value="/login.jsp"/>
		<property name="unauthorizedUrl" value="/unauthorized.jsp"/>
		<property name="filterChainDefinitions">
			<value>
				/css/** = anon
				/js/** = anon
				/images/** = anon
				/validatecode.jsp* = anon
				/login.jsp* = anon
				/userAction_login.action = anon
				/page_base_staff.action = perms["staff"]
				/** = authc
				<!--/** = authc-->
			</value>
		</property>
	</bean>
	<!--开启自动代理,并且将代理代理模式设置为cjlib-->
	<bean id="defaultAdvisorAutoProxyCreator" class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator">
		<!--设置成cglib方式-->
		<property name="proxyTargetClass" value="true"></property>
	</bean>
```
![shiro各种过滤器简写](http://img.blog.csdn.net/20170109205450918?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzQ1MTM0Nw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



#  shiro的使用 		

1. 在web.xml中引入用于创建shiro框架的过滤器
web.xml中:DelegatingFilterProxy=>spring整合shiro   
`注意引入的位置:要在struts核心过滤器的前面,StrutsPrepareAndExcutFilter,不然,所有请求会通过struts过滤器获直接访问得到,shiro的过滤器将不会起到作用`    

2. 在Spring中整合shiro   
  2.1).  shiro框架过滤器:**ShiroFilterFactoryBean** 需要声明那些过滤器,那些资源需要匹配那些过滤器,采用url拦截方式进行的路径对应的拦截器    
  2.2).  配置安全管理器:**DefaultWebSecurityManager** 需要注入 自定义的Realm bean对象     

  ```xml
  <!--配置一个shiro框架的过滤器工厂bean,用于创建shiro框架的过滤器-->
	<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
		<property name="securityManager" ref="securityManager"/>
		<property name="loginUrl" value="/login.jsp"/>
		<property name="unauthorizedUrl" value="/unauthorized.jsp"/>
		<property name="filterChainDefinitions">
			<value>
				/css/** = anon
				/js/** = anon
				/images/** = anon
				/validatecode.jsp* = anon
				/login.jsp* = anon
				/userAction_login.action = anon
				/page_base_staff.action = perms["staff"]
				/** = authc
				<!--/** 表示所有/下所有路径,包括下面的所有路径-->
        <!--/validatecode.jsp*
         表示所有除了validatecode.jsp,还包括jsp后追加其他内容的.如validatecode.jsp?'+Math.random();防止验证码读取缓存
			</value>
		</property>
	</bean>
	<!--开启自动代理,并且将代理代理模式设置为cjlib
  动态代理分为两类
  基于jdk 创建的类必须要实现一个接口,这是面向接口的动态代理   
  基于cjlib 创建的类不能用final修饰
-->
	<bean id="defaultAdvisorAutoProxyCreator" class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator">
		<!--设置成cglib方式-->
		<property name="proxyTargetClass" value="true"></property>
	</bean>
	<!--定义aop通知+切入点-->
	<bean id="authorizationAttributeSourceAdvisor" class="org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor"></bean>

	<!--注入安全管理器-->
	<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
		<property name="realm" ref="bosRealm"></property>
		<property name="cacheManager" ref="ehCacheManager"></property>
	</bean>
  ```

3. 在登陆认证的方法中加入subject `controller中的login方法`
```java
public String login(){
Subject subject = SecurityUtils.getSubject();
  //创建一个用户名密码令牌
  AuthenticationToken token = new UsernamePasswordToken(getModel().getUsername(), MD5Utils.md5(
          getModel().getPassword()));
  try {
    //认证
      subject.login(token);
  } catch (Exception e) {
      this.addActionError("用户名或者密码错误");
      return LOGIN;
  }
  /*当通过认证,跳入主页*/
  User user = (User) subject.getPrincipal();
  /*将用户信息存入session*/
  ServletActionContext.getRequest().getSession().setAttribute("currentUser", user);
  /*返回主页*/
  return "";
}
```  

4. 自定义Realm(用于权限的具体实施,即认证和授权)一般实现Realm接口的 **AuthorizingRealm** 实例    
4.1实现认证 重写doGetAuthenticationInfo方法
必须继承*AuthorizingRealm*    
在需要交付给spring生成,并需要在安全注册管理器中注入属性Realm


```java
protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {

UsernamePasswordToken mytoken = (UsernamePasswordToken) token;
       String username = mytoken.getUsername();
       DetachedCriteria dc = DetachedCriteria.forClass(User.class);
       dc.add(Restrictions.eq("username",username));
       List<User> list = userDao.findByCriteria(dc);
       if(list != null && list.size() >0){
           User user = list.get(0);
           String dbPassword = user.getPassword();
           AuthenticationInfo info = new SimpleAuthenticationInfo(user,dbPassword,this.getName());
           return info;
       }else{
           return null;
       }
   }
```
4.2实现授权 重写doGetAuthorizationInfo方法



```java
protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
/*获的简单授权对象,用于授权的*/
  SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
    /*授权staff权限*/
    //info.addStringPermission("staff");
    //步骤获得授权对象,获得当前用户,获得当前用户的权限(若为admin即授予所有权限),当前用户授权
    //获得对象
    User user = (User)principals.getPrimaryPrincipal();
    List<Function> fList = null;
    //获得权限
    if(user.getUsername().equals("admin")){
        fList = functionDao.findAll();
    }else{
        fList = functionDao.findFunctionByUserId(user.getId());
    }
    //授予权限
    for(Function f : fList){
        info.addStringPermission(f.getCode());
}
```			

## 关于Shiro中使用 **encache**     
1.引入包   
`在spring配置文件中配置以下`  
2.配置文件ehcache.xml   
```xml
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="../config/ehcache.xsd">
    <defaultCache
            maxElementsInMemory="10000"
            eternal="false"
            timeToIdleSeconds="120"
            timeToLiveSeconds="120"
            overflowToDisk="true"
            maxElementsOnDisk="10000000"
            diskPersistent="false"
            diskExpiryThreadIntervalSeconds="120"
            memoryStoreEvictionPolicy="LRU"
            />
</ehcache>
 <!--eternal是否永久有效-->
```
3.引入缓存管理器**EhCacheManager**(shiro包中的),并设置配置文件;     
4.将缓存管理器注入安全管理器**DefaultWebSecurityManager**
```xml
<!--注册安全管理器-->
<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
		<property name="realm" ref="bosRealm"></property>
		<property name="cacheManager" ref="ehCacheManager"></property>
	</bean>
	<bean id="bosRealm" class="org.yao.bos.web.action.realm.BOSRealm"></bean>
	<!--注入缓存管理器-->
	<bean id="ehCacheManager" class="org.apache.shiro.cache.ehcache.EhCacheManager">
		<property name="cacheManagerConfigFile" value="classpath:ehcache.xml"></property>
	</bean>
```

