---
title: Spring Boot中使用@Async实现异步调用
date: 2018-01-20 10:59:44
tags: [SpringBoot]
categories: SpringBoot
---




> 在实际项目中，经常需要用到异步处理，大部分情况我们都会使用定义一个线程（Thread），通过ExecutorService来调度，这样实现没有任何问题，那么有没有更简单方便的实现呢？Spring中提供了@Async注解，来简化异步操作。
<!-- more -->

@Async的定义
---
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Async {
    String value() default "";
}
```

Async的Target包含Method和Type，也就是可以定义在类或者方法上，那么@Async的工作原理是什么样的呢?

> spring在扫描bean的时候会扫描方法上是否包含@async的注解，如果包含的，spring会为这个bean动态的生成一个子类，我们称之为代理类,代理类继承我们所写的bean的，然后把代理类注入进来，那此时，在执行此方法的时候，会到代理类中，代理类判断了此方法需要异步执行，就不会调用父类
(我们原本写的bean)的对应方法。spring自己维护了一个队列，他会把需要执行的方法，放入队列中，等待线程池去读取这个队列，完成方法的执行，
从而完成了异步的功能。我们可以关注到再配置task的时候，是有参数让我们配置线程池的数量的。因为这种实现方法，***所以在同一个类中的方法调用，添加@async注解是失效的！，原因是当你在同一个类中的时候，方法调用是在类体内执行的，spring无法截获这个方法调用***。

@Async的使用
---
 使用@Async很简单，只需要在方法上加上注解即可，方法可以定义2种，有返回值和没有返回值得，有的时候我们还是希望获取该异步任务的返回值
```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.AsyncResult;
import org.springframework.stereotype.Component;

import java.util.Random;
import java.util.concurrent.Future;

import lombok.extern.slf4j.Slf4j;


/**
 * 基于Async注解的异步任务，需要开启@EnableAsync
 * Author: gaowenming
 * Description:
 * Date: Created in 20:49 2017/7/2.
 */
@Component
@Slf4j
public class AsyncTask {
    private static Random random = new Random();

    @Async
    public Future<String> task1() throws Exception {
        System.out.println("开始做任务一");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        System.out.println("完成任务一，耗时：" + (end - start) + "毫秒");
        return new AsyncResult<>("任务一完成");
    }
    
    @Async("smartExecutor")
    public void taskAsync1() throws Exception {
        Thread.sleep(1000);
        for (int i = 0; i < 10; i++) {
            log.info("##########################" + i);
        }
    }
}

```
上面的实例中，我们定义了2个方法，在执行异步操作时，我们还可以指定线程池，@Async("smartExecutor")，smartExecutor是我们定义的线程池
```java

/**
 * 定义线程池
 */
@Configuration
@EnableAsync
public class ThreadExecutorConfig {

    /**
     * Set the ThreadPoolExecutor's core pool size.
     */
    private static final int corePoolSize = 10;
    /**
     * Set the ThreadPoolExecutor's maximum pool size.
     */
    private static final int maxPoolSize = 200;
    /**
     * Set the capacity for the ThreadPoolExecutor's BlockingQueue.
     */
    private static final int queueCapacity = 10;

    @Bean
    public Executor smartExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(queueCapacity);
        executor.setThreadNamePrefix("smartExecutor-");
        executor.initialize();
        return executor;
    }

}
```
这里需要注意的是，在SpringBoot中，要启用@Async注解，还需要在config上加上@EnableAsync注解，表示启用@Async注解

线程池和异步方法定义好之后，接下来就是调用，我们可以和普通方法一样
```java
@RestController
@RequestMapping("api/task/")
@Slf4j
@Api
public class TaskController extends BaseController {

    @Autowired
    private AsyncTask asyncTask;

    @RequestMapping(value = "/async", method = RequestMethod.GET)
    public BaseJsonResult task1() throws Exception {
        log.info("test async................");
        asyncTask.task1();
        asyncTask.taskAsync1();
        return successNullDataResult();

    }

}

```
是不是很方便，在需要异步操作的时候，终于不用再自己创建Thread了，而且也可以在方法上加上参数，和普通方法一样，很大的增加了代码的可读性。

注意的问题 
---
 在实际的开发中，其实我们可以把需要异步操作的方法定义公共的类中，也可以给不同的操作指定不同的线程池，那么如果在异步操作中发生异常该如何处理呢？Spring提供了AsyncUncaughtExceptionHandler这个Handler来处理，我们可以在发生异常的时候，根据不同操作的定义不同的处理机制。


 


 
