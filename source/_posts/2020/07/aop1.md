---
title: AOP
date: 2020-07-12 10:35:59
tags:
- AOP
categories:
- spring
- AOP
---

### 前言

  本章主要总结下AOP的原理及实现。
<!-- more -->

### 代理模式

  AOP基于代理模式的设计思想，先来看看什么是代理模式

{% img  /image/aop/aop1.png  '"代理模式的结构"' %}

{% codeblock lang:java %}
/**
 * 服务接口
 *
 * @author RF
 * @date 2020/7/13
 */
public interface ServiceInterface {
    void show(String msg);
}

/**
 * 真实服务
 *
 * @author RF
 * @date 2020/7/13
 */
public class RealService implements ServiceInterface {
    private static final Logger logger = LoggerFactory.getLogger(RealService.class);

    @Override
    public void show(String msg) {
        logger.info("realService info:{}", msg);
    }
}

/**
 * 代理服务
 *
 * @author RF
 * @date 2020/7/13
 */
public class ProxyService implements ServiceInterface {
    private final Logger logger = LoggerFactory.getLogger(ProxyService.class);

    private RealService realService;

    @Override
    public void show(String msg) {
        logger.info("proxy service：{}", msg);
        realService.show(msg);
    }
}
{% endcodeblock%}

  代理类与被代理类实现相同的接口，这样保证两者的结构一致性。同时两者又是组合关系，代理类中包含被代理类的实例。

  代理模式可以在不修改原有代码的前提下，对其进行扩展，符合开放封闭原则，但其缺点是要新建许多代理类。
  
### 动态代理

  上述实现代理模式的方法也被称为静态代理，其缺点显而易见，所以就有了动态代理的这种方式去实现。

#### 基于JDK的动态代理

{% codeblock lang:java %}
/**
 * 实现InvocationHandler的代理类
 *
 * @author RF
 * @date 2020/7/13
 */
public class JdkProxy<T> implements InvocationHandler {
    private static final Logger logger = LoggerFactory.getLogger(JdkProxy.class);
    private T realService;

    public JdkProxy(T realService) {
        this.realService = realService;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        logger.info("parameter：{}", method);
        logger.info("args:{}", args[0]);
        method.invoke(realService, args[0]);
        return null;
    }
}

public void main(String[] args){
    RealService realService = new RealService();
    JdkProxy<RealService> handler = new JdkProxy<>(realService);
    ServiceInterface serviceInterface = (ServiceInterface) Proxy.newProxyInstance(
            ServiceInterface.class.getClassLoader(),
            new Class[]{ServiceInterface.class},
            handler);
    serviceInterface.show("hello world");
}

// output:
// parameter：public abstract void com.example.demo.proxy.ServiceInterface.show(java.lang.String)
// args:hello world
// realService info:hello world
{% endcodeblock %}

  这种方式的好处是针对接口的某种扩展只需新建一个实现了*InvocationHandler* 的代理类，然后调用Proxy的方法就能实现。而不需要像静态代理一样对每一个接口的实例新建一个对应的代理类。

#### 基于CGLIB的动态代理
  
  待补充


### 注解方式实现AOP

  在spring boot中，我们可以更加方便的使用注解的形式实现AOP。

{% codeblock lang:java %}
// 切面类
@Aspect
@Component
public class MyAspect {
    private static final Logger logger = LoggerFactory.getLogger(MyAspect.class);

    @Pointcut("@annotation(com.example.demo.aop.MyLogger)")
    public void myPointCut() {
    }

    @Before("myPointCut()")
    public void beforeMethod(JoinPoint joinPoint) {
        logger.info("before time:{}", LocalDateTime.now());
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        logger.info("method name:{}", method.getName());
        MyLogger myAnnotation = method.getAnnotation(MyLogger.class);
        if (null != myAnnotation) {
            logger.info("annotation name:{}", myAnnotation.name());
        }
        //获取接口入参
        Object[] args = joinPoint.getArgs();
        StringBuilder sb = new StringBuilder();
        if (null != args) {
            for (Object o : args) {
                sb.append(o).append(",");
            }
        }
        // 输出接口入参
        logger.info("parameter:{}", sb);
    }

    @After("myPointCut()")
    public void afterMethod() {
        logger.info("after time:{}", LocalDateTime.now());
    }
}

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface MyLogger {

    String name() default "";
}

// 使用注解标记目标接口
@GetMapping(value = "/test/send")
@MyLogger(name = "hello world")
public void send(@RequestParam("message") String message,
                 @RequestParam("userId") Integer userId) {
    logger.info("do something·······:{},{}", message, userId);
}
{% endcodeblock %}

{% img /image/aop/aop2.png '"调用结果"' %}

### 小结

      AOP是一种面向切面编程的思想，他的目的是将不影响业务逻辑的共同操作给提出来，将其与主逻辑分
    离，一方面能够简化代码，一方面达到解耦的目的。
      这些分离出来的操作又按照需求插入到主逻辑的代码间隙中，所以被形象的成为面向切面编程。
      其应用场景很多，最常见的就是类似上述例子中接口入参的日志输出，这样可以不用在每个接口开头去写
    一行日志输出的代码了。
      而动态代理的实现原理可以理解为对字节码的增强，其会在运行时对目标类的字节码进行修改，增加指定
    的切面操作的内容。

### 参考

>  [代理设计模式](https://refactoringguru.cn/design-patterns/proxy)
>  [动态代理-廖雪峰的官方网站](https://www.liaoxuefeng.com/wiki/1252599548343744/1264804593397984#0)
>  [java动态代理实现与原理详细分析](https://www.cnblogs.com/gonjan-blog/p/6685611.html)
