---
title: "openstack4j 中的 IoC 设计"
date: 2018-04-23
draft: false
tags:
- dp
categories:
- dp
---


## openstack4j 简介

openstack4j 是一个开源的 Java OpenStack SDK. 更多信息请了解其 [官网](http://openstack4j.com/).

源码地址：[https://github.com/ContainX/openstack4j.git](https://github.com/ContainX/openstack4j.git)

## openstack4j 中的 IoC

关于 DI 和 IoC 的区别实为仁者见智的问题，本文不会区分这两个概念。

openstack4j 并没有采用第三方的 DI/IoC 实现，如 Guice 等，而是使用很少的代码实现了简单、优雅的 IoC 管理。

### SPI

[SPI](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html) 是一种服务发现机制。简单来讲，服务提供者在 META-INF/services/ 目录有一个以服务接口命名的文件，文件中写明实现类。

jdk 提供一个工具类 java.util.ServiceLoader 用于载入实现。

在 openstack4j/core/src/main/java/org/openstack4j/api/Apis.java 中，可以看到有一个 APIProvider 类型的静态成员:

```java
    private static final APIProvider provider = initializeProvider();
    // ...
    private static APIProvider initializeProvider() {
        // No need to check for emptiness as there is default implementation registered
        APIProvider p = ServiceLoader.load(APIProvider.class, Apis.class.getClassLoader()).iterator().next();
        p.initialize();
        return p;
    }
```

APIProvider 在初始化时，从 openstack4j/core/src/main/resources/META-INF/services/ 下的 `org.openstack4j.api.APIProvider` 的文本文件中读取实现类 `org.openstack4j.openstack.provider.DefaultAPIProvider`.

### APIProvider 设计

APIProvider 提供两个功能：

- API实现的注册（绑定）
- 实例对象的获取

这两点也是一个 DI/IoC 实现的核心。

APIProvider 的源码：

位置： `openstack4j/core/src/main/java/org/openstack4j/api/APIProvider.java`


```java
package org.openstack4j.api;

/**
 * To keep our dependencies simple in the current Openstack4J, we utilize ServiceLoader to load a provider who is responsible
 * for loading the implementation for any of the defined API interfaces.  This allows us to avoid pulling in extra 3rd party 
 * dependencies such as Guice, Inject, etc.  It also allows us flexibility on the provider which may be overriden and choose to bind
 * modules and simple delegate out the {@link #get(Class)} calls
 * 
 * @author Jeremy Unruh
 */
public interface APIProvider {

	/**
	 * Called after the APIProvider is constructed.  This allows the provider to pre-initialize or bind any interface implementations if desired
	 */
	void initialize();
	
	/**
	 * Gets the implementation for the specified interface type
	 *
	 * @param <T> the Openstack4j API type
	 * @param api the API interface
	 * @return the implementation for T
	 * @throws ApiNotFoundException if the API cannot be found
	 */
	<T> T get(Class<T> api);
}
```

OpenStack 的 API, 对客户端 SDK 来说是稳定的，因此 APIProvider 中不需要暴露 bind 方法，而是在 initialize 中将已知的 OpenStack API 全部做初始化绑定。

get 接口用于根据 interface 的 class 获取其实现类的实例。


### DafaultAPIProvider 实现

源码路径：

`openstack4j/core/src/main/java/org/openstack4j/openstack/provider/DefaultAPIProvider.java`

所有 OpenStack API 的实现都注册在此。从 import 语句可以看出 compute, identity, sahara, image, telemetry, trove 等 OpenStack 各个组件的 API 都包含在内。

```java
    private static final Map<Class<?>, Class<?>> bindings = Maps.newHashMap();
    private static final Map<Class<?>, Object> instances = Maps.newConcurrentMap();
```

DafaultAPIProvider 的内部使用了两个 Map 来保存实现类注册表和实例缓存表。

- bindings 是 API interface class 和 API implementation class 的映射
- instances 是一个缓存，将按需实例化的对象放入其中，再次读取时可直接返回

bindings 使用了 HashMap，而没有使用 ConcurrentMap 的原因是，bind 方法不暴露，全部绑定在 initialize 中完成， 没有并发访问的场景。而 instances 使用了 ConcurrentMap 是因为 get 会有并发访问场景。


initialize 中完成所有 API 实现类的注册：

```java
@Override
    public void initialize() {
        bind(org.openstack4j.api.identity.v2.IdentityService.class, org.openstack4j.openstack.identity.v2.internal.IdentityServiceImpl.class);
        bind(TenantService.class, TenantServiceImpl.class);
        bind(ServiceManagerService.class, 
        bind(CronTriggerService.class, CronTriggerServiceImpl.class);
        // ...
    }
```


get 的实现：

```java
@Override
    public <T> T get(Class<T> api) {
        if (instances.containsKey(api))
            return (T) instances.get(api);

        if (bindings.containsKey(api)) {
            try {
                T impl = (T) bindings.get(api).newInstance();
                instances.put(api, impl);
                return impl;
            } catch (Exception e) {
                throw new ApiNotFoundException("API Not found for: " + api.getName(), e);
            }
        }
        throw new ApiNotFoundException("API Not found for: " + api.getName());
    }
```

- 首先检查 instances 缓存，如果有就直接返回；

- 如果没有，检查是否已注册实现类
	- 如果注册了实现类，使用**反射**实例化之，并注册到缓存中，然后返回；
	- 如果没有注册实现类，抛出异常。


## 总结

openstack4j 的代码非常清晰、易懂； 其 IoC 设计遵循简单、实用的原则，够用即可。

BTW，openstack4j 的 SDK 设计大量采用 builder 模式，支持链式调用，使用起来非常流畅：

```java
// Create a Server Model Object
Server server = Builders.server()
                        .name("Ubuntu 2")
                        .flavor("large")
                        .image("imageId")
                        .build();

// Boot the Server
Server server = os.compute().servers().boot(server);
```