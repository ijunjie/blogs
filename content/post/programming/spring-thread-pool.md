---
title: "Spring Thread Pool"
date: 2018-07-29T13:04:35+08:00
draft: false
tags:
- threadpool
- spring
categories:
- programming
---

Spring 的 ThreadPoolTaskExecutor 对 JDK 的 ThreadPoolExecutor 做了很多封装，使用较为简单。

一个显著区别是，Spring 的 ThreadPoolTaskExecutor 不能 shutdown，也不能 awaitTermination. 这是因为 ThreadPoolTaskExecutor 是公用的。

Spring 中配置线程池 Bean 示例：

```java
@Configuration
@EnableAsync
public class ExecutorConfig {

    @Bean
    public Executor kaproIOIntensiveExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(16);
        executor.setMaxPoolSize(32);
        executor.setQueueCapacity(100000);
        executor.setThreadNamePrefix("myexecutor-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

也可以暴露 JDK 的 ExecutorService:

```java
/**
 * For an alternative, you may set up a ThreadPoolExecutor instance directly
 * using constructor injection, or use a factory method definition that points to the
 * java.util.concurrent.Executors class.
 * This is strongly recommended in particular for common @Bean methods in
 * configuration classes, where this FactoryBean variant would force you to
 * return the FactoryBean type instead of the actual Executor type.
 */
@Bean
public FactoryBean<ExecutorService>  nativeExecutorService() {
    ThreadPoolExecutorFactoryBean nativeExecutor = new ThreadPoolExecutorFactoryBean();
    // ...
    return nativeExecutor;
}
```

Spring 的 AsyncResult 是可以转换为 CompletableFuture 的：

```java
@Async("kaproIOIntensiveExecutor")
public CompletableFuture<SquareResult> squareAsync(int i) {
    try {
        Thread.sleep(2000);
    } catch (InterruptedException e) {
        // 不成功的任务使用日志记录，或使用 Optional
        return null;
    }

    SquareResult result = new SquareResult(i, i * i);
    System.out.println(Thread.currentThread().getName() + ", ---" + result);
    return new AsyncResult(result).completable(); // 转换为 CompletableFuture
}
```

一个将 List<CompletableFuture> 转换为 CompletableFuture<List> 的方法：

```java
static <T> CompletableFuture<List<T>> sequence(List<CompletableFuture<T>> futures) {
    return CompletableFuture
            .allOf(futures.toArray(new CompletableFuture[futures.size()]))
            .thenApply(v -> futures.stream()
                    .map(CompletableFuture::join)
                    .collect(Collectors.<T>toList()));
}
```