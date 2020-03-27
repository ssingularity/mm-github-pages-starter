---
title: "SpringBoot开发相关"
date: 2019-09-18
categories:
  - Spring Boot
tags:
  - Java
  - Spring Boot
header:
  teaser: /assets/images/Spring.jpg
  image: /assets/images/Spring.jpg
---

### 在实习过程中学到了很多Spring相关的骚操作,现在总结如下：

- 使用AutowireCapableBeanFactory可以把那些我们自己new的对象，纳入spring控制，从而可以使得内部属性可以被autowired

- ControllerAdvice+ExceptionHandler来处理异常（ResponseEntityExceptionHandler）
runtimeExceptuin等于uncheckedException也即发生了不一定会死，所以不需要显式地去捕获也不需要声明throws
通过继承runtimeException得到runtimeException
通过继承Exception得到checkedEdception

- 在Spring Security中WebSecurityConfigurerAdapter用来声明Security的相关配置，包括AuthenticationProvider，Filter, url路径访问权限等, GlobalMethodSecurityConfiguration用来声明式给方法的访问提供权限认证（preAuthorize, PermissionEvaluator提供方法权限认证的具体实现

- 流处理中的distinct可以用来去重，但是需要实现对象的equals以及hashcode方法

- 使用@Configurable(autowire = Autowire.BY_TYPE)来对Entity依赖注入，同时使用@Transient告知jpa不用持久化，需要在应用程序配置中加入@EnableSpringConfigured @EnableTransactionManagement(mode = AdviceMode.ASPECTJ)，同时在启动项目时需要加上参数-javaagent:aspectjweaver-1.9.4.jar，其中jar包需要下载到本地

- 配置刷新会自动加载带有@ConfigurationProperties和@RefreshScope的Bean另外RefreshScope不会传递

- Redis分布式锁可以使用RedisLockRegistry,需要添加spring-integration-redis依赖，ZooKeeper则有对应的ZookeeperLockRegistry

- 可以使用Audit功能来自动审计，记录@CreatedBy @CreatedDate @LastModifiedBy @LastModifiedDate，其中创建者和修改者的名字可以通过Spring Security加上一个实现了AuditorAware的类来实现

- POST 访问 cfgserverIp/actuator/bus-refresh来使用RabbitMQ总线刷新各自配置

- Arthas可以用来做在线java性能检测和bug排查甚至可以热更新

- 当插件的groupId没有显示提供的时候maven会去settings.xml中的pluginGroup中去寻找，默认提供了org.apache.maven.plugins和org.codehaus.mojo

- public <T> T get(Class<T> key)从而可以通过传入一个泛形来做一些事情

- mvn dependency:tree可以查看项目所有的依赖以及间接依赖，使用idea的maven helper插件在pom.xml文件中点击Dependency Analyzer可以快速地定位依赖冲突以及间接依赖

- 可以使用nip.io仿照域名

- 通过Resources.getResource可以拿到在resource下的配置文件

- 使用PageRequest.of构建PageRequset，在Repository的find中加入PageRequest则可以开启分页

- 单例双重锁机制——查，加锁，再查，都匹配了才执行

- 可以使用okhttpclient创建websocketClient，灰常好用！

- 可以使用Gatling用来做负载测试

- 访问permmitAll的接口的时候，spring security会帮助用户生成一个Anoymonous User的Role

- 可以使用RateLimiter用来做限流（令牌桶，会在1s内均匀的分发令牌）

- Spring推荐在构造器上使用@Autowired而不是在Field上

- 所有被注解了如@Configuration、@Component的都会先调用其对应的构造函数来创建对象,同时构造函数中的参数会自动从Spring beanFactory容器中去找到适配的bean来传入

- ObjectProvider提供了更加宽泛的依赖注入，允许对应依赖并不存在，从而使得构造函数的扩展性更好，在AutoConfigure中大量地使用了多参函数配合ObjectProvider作为参数作为Configuration的构造函数

- @ConfiguarionProperties有三种方式可以激活:1.在类本身申明为@Component 2.在Configuraion文件中new一个对应的对象返回并声明为Bean 3. 使用@EnableConfigurationProperties（xxx.class），对于这3中方法，Properties具体的值都会在初始化之后由ConfiguraionPropertiesBindingPostProcessor类来绑定

- Bean声明周期：![Bean生命周期](/assets/images/lifecycle.jpg)
  1. 无参构造函数(如果是基于@Bean声明的话，就是@Bean修饰的方法)
  2. populateBean(对于AutoWired的属性进行注入)
  3. Aware相关的接口
  4. BeanPostProcessor.postProcessBeforeInitialization
  5. @PostConstruct修饰的方法
  6. InitializingBean.afterPropertiesSet
  7. initMethod声明的方法
  8. BeanPostProcessor.postProcessAfterInitialization

- Spring初始化Bean时先根据所有的Bean生成BeanDefinition列表，在BeanDefinition中会有Bean定义的信息包括DepenOn信息（这时候因为不需要初始化只是登记信息，所以Depend的Bean还没有被发现都没有关系），在所有的BeanDefinition都整理好后，更具BeanDefinition列表来初始化所有的Bean，这时候如果有DependOn则会先递归初始化依赖的Bean