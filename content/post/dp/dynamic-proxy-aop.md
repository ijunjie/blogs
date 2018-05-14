---
title: "动态代理和 AOP"
date: 2018-05-14
draft: false
tags:
- dp
- proxy
- aop
- spring
categories:
- dp
---

JDK 动态代理创建时比较快，运行时较慢；CGlib 动态代理正好相反，创建时慢，运行时快。一般选择在系统初始化时，使用 CGlib 代理，并托管到 ApplicationContext 中。

Spring AOP 经历了自己实现 Advice/Advisor 织入到采用 AspectJ 的演化过程, 实现了配置的大幅简化。

## JDK 动态代理

JDK 的动态代理较之于静态代理的好处是：interface 如果增加了方法，则目标实现类必然是要修改的，静态代理类被迫也要修改，而改用动态代理，则无需修改代理类。

JDK 的动态代理，获取代理类实例时，必须提供一个目标类的实例。

```java
Hello impl = new HelloImpl(); 
Object o = Proxy.newProxyInstance(
        Hello.class.getClassLoader(),
        impl.getClass().getInterfaces(),
        (proxy, method, args1) -> {
            System.out.println("before...");
            System.out.println("proxy hash: " + System.identityHashCode(proxy));
            // 闭包中用到一个目标类实例对象
            Object result = method.invoke(impl, args1);
            System.out.println("after...");
            return result;
        });
Hello proxy = Hello.class.cast(o);
proxy.hi("hello");

System.out.println("outer: " + System.identityHashCode(proxy));
```

## CGlib 动态代理

而 CGlib 的代理不需要有一个目标类的对象，并且是方法级别的代理。正因如此，在 MethodInterceptor 中，是拿不到代理对象的引用的，但却可以拿到目标类对象和目标类的方法。由此可推断，CGlib 会使用反射生成一个目标类对象。下面代码中，如果目标类没有无参构造方法，会报错。

```
Object o = Enhancer.create(
            HelloCGlibImpl.class,
            (MethodInterceptor) (o, method, objects, methodProxy) -> {
                System.out.println("aaa");
                Object result = methodProxy.invokeSuper(o, args);
                System.out.println("bbb");
                return result;
            });
HelloCGlibImpl proxy1 = HelloCGlibImpl.class.cast(o);
```

## Spring AOP

### Advice

Spring AOP 中除 `MethodBeforeAdvice` 和 `MethodReturningAdvice` 以及 `org.aopalliance.intercept.MethodInterceptor` 外，还有一个 `ThrowsAdvice`，可以实现对目标类方法抛出异常的拦截处理。

AOP 有一套完整而复杂的理论。对方法的增强叫做 `Weaving`, 对类的增强叫做 `Introduction`. `DelegatingIntroductionInterceptor` 可以实现动态实现接口。这样做其实并不好，首先给程序可读性带来困难，也不利于 IDEA 做静态代码分析。

### Advisor

至此，AOP 都是拦截目标类的全部方法。实际需要的是更加灵活的控制。此时，AOP 的术语又来了， `Pointcut`用于在目标类的诸多方法中选出想要拦截的那些。具有 `Pointcut` 的拦截器叫 `Advisor`, 不再是前面的 `Advice` 了。`Advisor` 只是对 `Advice` 和 `Pointcut`的封装而已。软件工程领域，总有一帮人擅长制造各种概念和术语，Martin Fowler 便是其中的代表人物。

Spring 提供了一些定制好的 Advisor，如

- `NamedMatchMethodPointcutAdvisor`, 顾名思义根据方法名称匹配
- `RegexpMethodPointcutAdvisor`, 根据正则表达式匹配目标方法
- `StaticMethodMatcherPointcutAdvisor`, 匹配静态方法
- `DefaultPointcutAdvisor`

### AutoProxyCreator

Spring 提供了自动生成代理的配置：只能匹配目标类的 `BeanNameAutoProxyCreator` 和 可以进一步匹配方法的 `DefaultAdvisorAutoProxyCreator`.

即便可以自动生成代理，也仍然无法摆脱复杂的配置。试图用声明式配置解决问题的人，没有认识到一点，开发者使用更加富有表现力的编程语言时可以获得更多乐趣，而不是面向无聊的配置文件编程。试问，有谁愿意编写枯燥的 html 或 xml 呢？这就是 gradle 出现的动机之一，将编程语言和配置结合到一起。 sbt 早已如此。后来的 spring boot 也推荐使用 Java 配置。

### AspectJ

Spring 终于顺应历史潮流，采用了 AspectJ 来简化配置。在 AspectJ 的加持下，有两种方式拦截目标类的目标方法。无非是为了更有效的筛选目标类的目标方法而已。

- 使用 `@Aspect` 声明一个切面，使用形如 `@Around("execution(* com.foo.bar.*(...))")` 的注解声明拦截方法。
- 给目标类的目标方法打上自定义注解，然后在 Aspect中注明 `@Around("@annotation(com.MyTag)")`

除 Aspect 之外，还有 Before, After, Around, AfterThrowing, DeclareParents.