---
title: "SpringBoot系列（自己写starter）"
date: 2019-10-13
categories:
  - Spring Boot
tags:
  - Spring Boot
  - Java
header:
  teaser: /assets/images/AutoConfiguration.jpg
  image: /assets/images/AutoConfiguration.jpg
---

最近在开发一个基于微服务的后台应用，自然而然地就分了多个模块并对于一些在各个服务间需要复用的代码统一放在了公用模块中比如Common模块，Security模块。这些模块通过jar包的形式被其他模块所复用，但于此同时一个问题也就摆在了我的面前：如果被复用的模块中有些类是基于Spring的，那么其中的Configuration以及Bean该怎么才能被注册到使用模块的Spring容器中呢。

一个最简单的想法就是在使用模块中配置一下对应的被使用模块的包名，比如```@ComponentScan("xx.xxx.xx)```,就可以通过扫描的方式把被使用的模块的Bean给纳入控制了，当这个模块只是你自己个人开发的时候到还好说，但是如果你要拿去给别人复用，或者作为libraray难道还要别人专门去看一下你源码里面对应的包名，再自己去配置对应的```@ComponentScan```嘛？这无疑过于麻烦而且不透明，幸运的是SpringBoot本身就提供了对应的自动配置以及透明化的解决方案，而它本身的Springboot家族中的各个starter引用包也是通过相同的方式配置，在这里我也将通过一个简单的例子来演示如何编写自己的starter，不过在此之前，先让我们熟悉一下一些常用的注解。

## 常用注解
- ### @ConfigurationProperties
     
  标有 @ConfigurationProperties 的类的所有属性和配置文件中相关的配置项进行绑定。（默认从全局配置文件中获取配置值），绑定之后我们就可以通过这个类去访问全局配置文件中的属性值了。需要注意这里需要添加@Component的注解来自动注册自己或者@EnableConfigurationProperties(class=xxx.class)注解来主动注册自己，同时需要自己提供Getter和Setter（可以考虑直接用Lombok的@Data）

- ### @Import

  @Import 注解支持导入普通 java 类，并将其声明成一个bean。主要用于将多个分散的 java config 配置类融合成一个更大的 config 类

  @Import注解在4.2之前只支持导入配置类，4.2之后支持导入普通的java类，并将其声明成一个bean

  @Import三种使用方式：
  - 直接导入普通的Java类（自动注册到容器中）
  - 配合自定义的ImportSelector使用（导入多个类，自动注册到容器中）
  - 配合ImportBeanDefinitionRegister使用（手动注册bean到容器中）

- ### @Condition

  @Conditional 注释可以实现只有在特定条件满足时才启用一些配置，使用时需要传入一个实现了Condition的类，来作为是否满足特定条件的校验

  除了自定一Condition外，Spring还为我们扩展了一些常用的Condition注解

  |扩展注解|作用|
  |---|---|
  |ConditionalOnBean|容器中存在指定 Bean，则生效|
  |ConditionalOnMissingBean|容器中不存在指定 Bean，则生效|
  |ConditionalOnClass|系统中有指定的类，则生效|
  |ConditionalOnMissingClass|系统中没有指定的类，则生效|
  |ConditionalOnProperty|系统中指定的属性是否有指定的值|
  |ConditionalOnWebApplication|当前是web环境，则生效|

## 开始编写自己的starter

- ### 创建项目
  一般spring-boot-starter都会有两个jar包，一个是spring-boot-autoconfigure，一个是spring-boot-starter,所以需要一个父子项目来包含这两个jar，其中spring-boot-starter是作为主要发布的jar但是它只有一个pom文件，来声明依赖对应的spring-boot-autoconfigure，而在spring-boot-autoconfigure中则是实现了主要的自动配置以及逻辑的地方，当然如果逻辑不复杂，完全可以把两个合为一个

  除此之外，对于artifactId, Spring 官方 Starter通常命名为spring-boot-starter-{name} 如 spring-boot-starter-web，Spring官方建议非官方Starter命名应遵循{name}-spring-boot-starter的格式

  项目结构如下图所示：
  ![image](/assets/images/starter项目结构.png)

- ### 添加maven依赖
  ```
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-autoconfigure</artifactId>
  </dependency>
  ```
  注意其中 spring-boot-configuration-processor 的作用是编译时生成spring-configuration-metadata.json， 此文件主要给IDE使用，用于提示使用。如在intellij idea中，当配置此jar相关配置属性在application.yml， 你可以用ctlr+鼠标左键，IDE会跳转到你配置此属性的类中。

- ### 编写配置类
  ```
  @Data
  @ConfigurationProperties(prefix = "swagger.project")
  @Component
  public class SwaggerProperties {
      private String title = "";

      private String version = "version1.0";

      private String basePackage = "";

      private String description = "";
  }
  ```

- ### 编写自动配置类
  ```
  @Configuration
  @ComponentScan("cn.pipipan.springboot.swagger")
  public class SwaggerConfiguration {
    @Autowired
    private SwaggerProperties swaggerProperties;

    @Bean
    public Docket restApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("")
                .apiInfo(apiInfo(swaggerProperties.getTitle(), swaggerProperties.getVersion()))
                .useDefaultResponseMessages(true)
                .forCodeGeneration(false)
                .select()
                .apis(RequestHandlerSelectors.basePackage(swaggerProperties.getBasePackage()))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo(String title, String version) {
        return new ApiInfoBuilder()
                .title(title)
                .description(swaggerProperties.getDescription())
                .version(version)
                .build();
    }
  }
  ```
  需要注意的是，因为SwagerProperties哪怕注解了@Component也不会被自动地纳入使用项目中，因为使用项目不会扫描这个包，所以这里主动配置了对应的扫描另外一个解决方案就是通过@EnableConfigurationProperties(SwaggerProperties.class)来解决，这个时候就不需要注解@Component了

- ### 添加spring.factories
  最后异步， 在resources/META-INF/下创建spring.factories文件，内容供参考下面：
  ```
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  cn.pipipan.springboot.swagger.SwaggerConfiguration
  ```
  在到这一步之前，其实对应的配置都不会被自动发现的，因为没有对应的包扫描或者主动注册，现在通过将对应的内容写入spring.factories中，也即autoConfiguration的类是什么，就可以实现主动地把自己注册到使用项目中去了，因为Spring boot项目在启动的时候，会主动地查看每个依赖下面的META-INF中的spring.factories来进行Bean的注册

## Gitee源码
[swagger-spring-boot-starter](https://gitee.com/ssingularity/swagger-spring-boot-starter)