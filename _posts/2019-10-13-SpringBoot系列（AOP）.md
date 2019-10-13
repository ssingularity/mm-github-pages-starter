---
title: "SpringBoot系列（AOP）"
date: 2019-10-13
categories:
  - Spring Boot
tags:
  - Java
  - Spring Boot
header:
  teaser: /assets/images/AOP.jpg
---
![image](/assets/images/AOP.jpg)
在我看来Spring最重要的两大特性一个是IoC另外一个就是AOP了，虽然很早之前就接触过AOP的相关概念，但是一直没有在项目中运用起来，这次决定一定要使用一下这个特性并梳理了这篇博客来记录一下一些相关概念

## AOP术语定义

1、横切关注点

对哪些方法进行拦截，拦截后怎么处理，这些关注点称之为横切关注点

2、切面（aspect）

类是对物体特征的抽象，切面就是对横切关注点的抽象

3、连接点（joinpoint）

被拦截到的点，因为Spring只支持方法类型的连接点，所以在Spring中连接点指的就是被拦截到的方法，实际上连接点还可以是字段或者构造器

4、切入点（pointcut）

对连接点进行拦截的定义

5、通知（advice）

所谓通知指的就是指拦截到连接点之后要执行的代码，通知分为前置、后置、异常、最终、环绕通知五类,有时候也被称为增强

6、目标对象

代理的目标对象

7、织入（weave）

将切面应用到目标对象并导致代理对象创建的过程

8、引入（introduction）

在不修改代码的前提下，引入可以在运行期为类动态地添加一些方法或字段

## AOP工作重点
1. 如何通过切点（Pointcut）和增强（Advice）定位到连接点（Jointpoint）上

2. 如何在增强（Advice）中编写切面的代码

## 添加maven依赖
```
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
```

## 创建切面类
```
@Aspect
@Component
@Slf4j
public class Aop {
    @Pointcut("execution(* com.sjtu.project.test.TestApplication.testAop(..))")
    public void pointcut() {

    }

    @Before("pointcut()")
    public void begin() {
      log.info("before");
    }

    @After("pointcut()")
    public void after() {
        log.info("after");
    }

    @AfterReturning(value = "pointcut()", returning = "ret")
    public void afterReturning(Object ret) {
        log.info("return: " + ret);
    }
    
    @AfterThrowing(value = "pointcut()")
    public void afterThrowing(JoinPoint jp) {
        log.info("afterThrow");
    }

    @Around("pointcut()")
    public void around(ProceedingJoinPoint pjp) {
        try {
            Object o = pjp.proceed();
            log.info("around");
        }
        catch (Throwable e) {
            e.printStackTrace();
        }
    }
}
```
大致流程就是:
### 先通过@PointCut和定义声明对应的切点
在这里给出一些常用的切点定义：

- 任意公共方法的执行：

  execution（public * *（..））

- 任何一个名字以“set”开始的方法的执行：

  execution（* set*（..））

- AccountService接口定义的任意方法的执行：

  execution（* com.xyz.service.AccountService.*（..））

- 在service包中定义的任意方法的执行：

  execution（* com.xyz.service.*.*（..））

- 在service包或其子包中定义的任意方法的执行：

  execution（* com.xyz.service..*.*（..））

- 在service包中的任意连接点：

  within（com.xyz.service.*）

- 在service包或其子包中的任意连接点：

  within（com.xyz.service..*）

- 任何一个只接受一个参数，并且运行时所传入的参数是Serializable 接口的连接点:

  args（java.io.Serializable）

- 任何一个执行的方法有一个 @Transactional 注解的连接点:

  @annotation（org.springframework.transaction.annotation.Transactional）

### 再给出对于切点需要增强的功能

### 最后通过@Before等切面注解加advice和pointcut粘合起来成切面
这里给出一些常用的切面注解：

- @Before 标识一个前置增强方法，相当于BeforeAdvice的功能

- @AfterReturning 后置增强，相当于AfterReturningAdvice，方法退出时执行

- @AfterThrowing 异常抛出增强，相当于ThrowsAdvice

- @After final增强，不管是抛出异常或者正常退出都会执行

- @Around 环绕增强，相当于MethodInterceptor

方法参数说明：

除了@Around外，每个方法里都可以加或者不加参数JoinPoint，JoinPoint里包含了类名、被切面的方法名，参数等属性，可供读取使用。

@Around参数必须为ProceedingJoinPoint，pjp.proceed相应于执行被切面的方法。

@AfterReturning里可以加returning = “xxx”，然后被注解方法的参数中通过Object xxx来引用joinpoint对应的方法的返回值

@AfterThrowing可以加throwing = “xxx”，然后被注解方法的参数中通过Exception xxx来引用joinpoint对应的方法的异常信息
