---
title: "SpringBoot系列（SpringSecurity）"
date: 2019-10-15
categories:
  - SpringBoot
tags:
  - SpringBoot
  - SpringSecurity
header:
  teaser: /assets/images/Spring-Security.jpg
---
![image](/assets/images/Spring-Security.jpg)
## 1.核心组件
### 1.1 SecurityContextHolder
  SecurityContextHolder用于存储安全上下文（security context）的信息，包括当前用户是谁，他拥有哪些角色权限，这些保存在SecurityContextHolder中
  获取当前用户信息的代码如下：
  ```
  Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();

  if (principal instanceof UserDetails) {
    String username = ((UserDetails)principal).getUsername();
  } 
  else {
    String username = principal.toString();
  }
  ```
  其中getContext()获得了对应的上下文，getAuthentication()获得了对应的Authentication信息，getPrincipal()返回了身份信息，它可能是UserDetails也可能就是单纯的用户名什么，具体视代码而定

### 1.2 Authentication
  Authentication顾名思义就是对应的保存鉴权信息的类，源码如下：
  ```
  public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities(); // 权限

    Object getCredentials();// 在认证过后通常会被移除，用于保障安全

    Object getDetails();// 细节信息，web 应用中的实现接口通常为 WebAuthenticationDetails，它记录了访问者的 ip 地址和 sessionId 的值。

    Object getPrincipal();// 身份信息

    boolean isAuthenticated();// 是否已经鉴权

    void setAuthenticated(boolean var1) throws IllegalArgumentException;
  }
  ```
  这里有点意思的是Authentication继承了Principal，但它还有一个对应的getPrincipal()的函数接口，但这两个并没有太大联系，getPrincipal()更多返回的是身份信息，像用户名什么的，当然有时候也有可能返回UserDetails，它也是Authentication的一个实现类，这就比较搞了

### 1.3 身份认证流程
  1. 用户名和密码被过滤器其获取到，封装成Authentication，通常情况下是UsernamePasswordAuthenticationToken这个实现类
  2. AuthenticationManager身份管理器负责验证这个Authentication
  3. 认证成功后，AuthenticationManager返回一个被填充了信息的Authentication实例
  4. SecurityContextHolder 安全上下文容器将第 3 步填充了信息的 Authentication，通过 SecurityContextHolder.getContext().setAuthentication(…) 方法，设置到其中

### 1.4 AuthenticationManager
  这个解释起来就比较麻烦了，因为这里用了一些设计模式，但是简单的说就是AuthenticationManager就是一个用来负责验证Authentication的身份管理器的抽象接口，它是认证相关的核心接口，也是发起认证的出发点，因为在实际需求中，我们可能会允许用户使用用户名 + 密码登录，同时允许用户使用邮箱 + 密码，手机号码 + 密码登录，甚至，可能允许用户使用指纹登录（还有这样的操作？没想到吧），所以说 AuthenticationManager 一般不直接认证，AuthenticationManager 接口的常用实现类 ProviderManager 内部会维护一个 List<AuthenticationProvider> 列表，存放多种认证方式，实际上这是委托者模式的应用（Delegate）。也就是说，核心的认证入口始终只有一个：AuthenticationManager，不同的认证方式：用户名 + 密码（UsernamePasswordAuthenticationToken），邮箱 + 密码，手机号码 + 密码登录则对应了三个 AuthenticationProvider

### 1.5 DaoAuthenticationProvider
  DaoAuthenticationProvider是最常用的一个AuthenticationProvider,顾名思义，Dao正是数据访问层的缩写，也就是从数据库读取用户信息（UserDetailss)然后和之前封装好的UsernameAndPasswordAuthenticationToken进行对比校验，如果成了，就代表校验成功

### 1.6 UserDetails和UserDetailsService
  在1.5中也稍微提到了这两个概念，其实它们就是从DaoAuthenticationProvider中衍生出来的概念，UserDetails代表了最详细的用户信息，我们完全可以让我们的实体类去实现它，UserDetailsService则是DaoAuthenticationProvider会调用的一个对应接口，所以我们也需要实现它：
  ```
  public interface UserDetailsService {
   UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
  }
  ```

### 1.7 架构概览图

![image](/assets/images/spring-security-architecture.png)

## 2.配置文件
### 2.1 http配置
```
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.cors()
      .and()
        .csrf().disable()
        .addFilter(new JWTBasicFilter(authenticationManager()))
      .and()
        .formLogin()
        .loginProcessingUrl("/login")
        .successHandler(authenticationSuccessHandler)
        .failureHandler(authenticationFailureHandler)
      .and()
        .logout()
        .logoutUrl("/logout")
      .and()
        .authorizeRequests()
        .antMatchers("/user").authenticated()
        .anyRequest().permitAll();
}
```
首先是cors和csrf控制了对应的跨域访问的一些配置
然后是addFilter，这里是加入一个过滤JWT信息并根据是否有JWT信息来鉴权的的Filter，它继承了BasicAuthenticationFilter,因为这个Filter继承了对应的Basic'AuthenticationFilter,所以它会以同样的Order被加入到FilterChain中去并覆盖默认的Filter
然后formLogin和logout分别控制了登录和登出的一些信息，像successHandler和failureHandler就是对应成功和失败以后的处理器，不过它们各自要实现对应的handler接口
最后是authorizeRequests就是对于Restful接口的一个接口级别的控制，如用例代码中就代表了/user是需要授权的，别的都不需要

### 2.2 鉴权配置
除了在2.2中看到的那样可以通过Filter集成对应鉴权Filter的方式来进行鉴权以外，另外一种方式就是直接通过AuthenticationManagerBuilder来进行配置，AuthenticationManager下的AuthenticationProvider应该默认使用了DaoAuthenticationProvider，代码如下：
```
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder);
}
```

### 2.3 @EnableWebSecurity
我们自己定义的配置类 WebSecurityConfig 加上了 @EnableWebSecurity 注解，同时继承了 WebSecurityConfigurerAdapter。你可能会在想谁的作用大一点，毫无疑问 @EnableWebSecurity 起到决定性的配置作用，它其实是个组合注解。
```
@Import({ WebSecurityConfiguration.class,
      SpringWebMvcImportSelector.class })
@EnableGlobalAuthentication 
@Configuration
public @interface EnableWebSecurity {
   boolean debug() default false;
}
```

## 3 FilterChain
### 3.1 加载了的Filters
加载了的Filters，我通过Console的日志提取了出来，它们如下所示：
```
org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter,
org.springframework.security.web.context.SecurityContextPersistenceFilter,
org.springframework.security.web.header.HeaderWriterFilter,
org.springframework.web.filter.CorsFilter
org.springframework.security.web.authentication.logout.LogoutFilter
org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter
org.springframework.security.web.authentication.ui.DefaultLogoutPageGeneratingFilter
com.sjtu.project.common.security.filter.JWTBasicFilter,
org.springframework.security.web.savedrequest.RequestCacheAwareFilter
org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter
org.springframework.security.web.authentication.AnonymousAuthenticationFilter
org.springframework.security.web.session.SessionManagementFilter
org.springframework.security.web.access.ExceptionTranslationFilter
org.springframework.security.web.access.intercept.FilterSecurityInterceptor
```
- SecurityContextPersistenceFilter 两个主要职责：请求来临时，创建 SecurityContext 安全上下文信息，请求结束时清空 SecurityContextHolder。
- HeaderWriterFilter (文档中并未介绍，非核心过滤器) 用来给 http 响应添加一些 Header, 比如 X-Frame-Options, X-XSS-Protection*，X-Content-Type-Options.
- CsrfFilter 在 spring4 这个版本中被默认开启的一个过滤器，用于防止 csrf 攻击，了解前后端分离的人一定不会对这个攻击方式感到陌生，前后端使用 json 交互需要注意的一个问题。
- LogoutFilter 顾名思义，处理注销的过滤器
- UsernamePasswordAuthenticationFilter 表单提交了 username 和 password，被封装成 token 进行一系列的认证，便是主要通过这个过滤器完成的，在表单认证的方法中，这是最最关键的过滤器。
- JWTBasicFilter 这个是我自己继承了BasicAuthenticationFilter的一个Filter，因为我之前说的原因它拥有了和BasicAuthenticationFilter一样的Order，它主要就是解析用户的Header中有没有携带对应JWT认证信息并且是否能被正常解析，如果可以的话就鉴权成功
- RequestCacheAwareFilter (文档中并未介绍，非核心过滤器) 内部维护了一个 RequestCache，用于缓存 request 请求
- SecurityContextHolderAwareRequestFilter 此过滤器对 ServletRequest 进行了一次包装，使得 request 具有更加丰富的 API
- AnonymousAuthenticationFilter 匿名身份过滤器，这个过滤器个人认为很重要，需要将它与 UsernamePasswordAuthenticationFilter 放在一起比较理解，spring security 为了兼容未登录的访问，也走了一套认证流程，只不过是一个匿名的身份。
- SessionManagementFilter 和 session 相关的过滤器，内部维护了一个 SessionAuthenticationStrategy，两者组合使用，常用来防止 session-fixation protection attack，以及限制同一用户开启多个会话的数量
- ExceptionTranslationFilter 直译成异常翻译过滤器，还是比较形象的，这个过滤器本身不处理异常，而是将认证过程中出现的异常交给内部维护的一些类去处理，具体是那些类下面详细介绍
- FilterSecurityInterceptor 这个过滤器决定了访问特定路径应该具备的权限，访问的用户的角色，权限是什么？访问的路径需要什么样的角色和权限？这些判断和处理都是由该类进行的。

### 3.2 SecurityContextPersistenceFilter
试想一下，如果我们不使用 Spring Security，如果保存用户信息呢，大多数情况下会考虑使用 Session 对吧？在 Spring Security 中也是如此，用户在登录过一次之后，后续的访问便是通过 sessionId 来识别，从而认为用户已经被认证。具体在何处存放用户信息，便是第一篇文章中提到的 SecurityContextHolder；认证相关的信息是如何被存放到其中的，便是通过 SecurityContextPersistenceFilter。在 4.1 概述中也提到了，SecurityContextPersistenceFilter 的两个主要作用便是请求来临时，创建 SecurityContext 安全上下文信息和请求结束时清空 SecurityContextHolder。顺带提一下：微服务的一个设计理念需要实现服务通信的无状态，而 http 协议中的无状态意味着不允许存在 session，但是其实这并不是真正意义上的session，因为它会随着每次请求的结束都自动清空所以这并不意味着 SecurityContextPersistenceFilter 变得无用，因为它还需要负责清除用户信息。在 Spring Security 中，虽然安全上下文信息被存储于 Session 中，但我们在实际使用中不应该直接操作 Session，而应当使用 SecurityContextHolder

### 3.3 UsernamePasswordAuthenticationFilter
表单认证是最常用的一个认证方式，一个最直观的业务场景便是允许用户在表单中输入用户名和密码进行登录，而这背后的 UsernamePasswordAuthenticationFilter，在整个 Spring Security 的认证体系中则扮演着至关重要的角色。
流程大致如下
![image](/assets/images/UsernamePasswordAuthenticationFilter.jpg)
再通过配置也可以看到UsernamePasswordAuthenticationFilter是被formLogin给开启的，而它也主要用了DaoAuthenticatoinProvider作为了默认的鉴权方式，当然我们也可以提供自己的AuthenticationProvider或者自己的UserDetailsService，当然注意需要同时提供对应的passwordEncoder并在用户注册的时候使用对应的encoder对于密码进行加密，目前比较好用的就是BCryptPasswordEncoder了

### 3.4 AnonymousAuthenticationFilter
匿名认证过滤器，可能有人会想：匿名了还有身份？我自己对于 Anonymous 匿名身份的理解是 Spirng Security 为了整体逻辑的统一性，即使是未通过认证的用户，也给予了一个匿名身份。而 AnonymousAuthenticationFilter 该过滤器的位置也是非常的科学的，它位于常用的身份认证过滤器（如 UsernamePasswordAuthenticationFilter、BasicAuthenticationFilter、RememberMeAuthenticationFilter）之后，意味着只有在上述身份过滤器执行完毕后，SecurityContext 依旧没有用户信息，AnonymousAuthenticationFilter 该过滤器才会有意义 —- 基于用户一个匿名身份。

### 3.5 ExceptionTranslationFilter
ExceptionTranslationFilter 异常转换过滤器位于整个 springSecurityFilterChain 的后方，用来转换整个链路中出现的异常，将其转化，顾名思义，转化以意味本身并不处理。一般其只处理两大类异常：AccessDeniedException 访问异常和 AuthenticationException 认证异常。

这个过滤器非常重要，因为它将 Java 中的异常和 HTTP 的响应连接在了一起，这样在处理异常时，我们不用考虑密码错误该跳到什么页面，账号锁定该如何，只需要关注自己的业务逻辑，抛出相应的异常便可。如果该过滤器检测到 AuthenticationException，则将会交给内部的 AuthenticationEntryPoint 去处理，如果检测到 AccessDeniedException，需要先判断当前用户是不是匿名用户，如果是匿名访问，则和前面一样运行 AuthenticationEntryPoint，否则会委托给 AccessDeniedHandler 去处理，而 AccessDeniedHandler 的默认实现，是 AccessDeniedHandlerImpl。所以 ExceptionTranslationFilter 内部的 AuthenticationEntryPoint 是至关重要的，顾名思义：认证的入口点。

配置的登录身份认证失败 handler failureHandler(..) 和 没有进行身份认证的异常 handler authenticationEntryPoint(..)，这两个有区别，前者是在认证过程中出现异常处理，后者是在访问需要进行身份认证的URL时没有进行身份认证异常处理。

## 4 自定义
自定义对应的鉴权验证方式有三种，按照简单到复杂分别是：
- 提供一个实现了UserDetailsService的类
- 提供一个自己的AuthenticationProvider
- 自己去实现UsernamePasswordAuthenticationFilter或者BasicAuthenticationFilter

在实现UsernamePasswordAuthenticationFilter的时候需要注意它的AuthenticationManager可以从外部传进来，也就是在配置类中注册自己的时候通过Bean的方式，将AuthenticationManager通过构造器来进行构造，同时它对应的AuthenticationProvider也在AuthentcationManagerBuilder中进行声明和注册，从而将Filter过程和实际鉴权分成了两块，从框架层面帮助达到了关注点分离