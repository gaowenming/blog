---
title: Spring Boot中使用@Scheduled执行定时任务
date: 2018-01-20 19:04:07
tags: Scheduled
categories: SpringBoot
---

> 日常工作中，有一些需要定时执行的任务，实现方式有Quartz，SpringTask，Quartz结合数据库使用，可以实现多节点的单实例执行。Spring Boot中也提供定时执行的解决方案@Scheduled。
<!-- more -->

创建任务
---

 - Spring Boot中要想使用@Scheduled，需要在Application中添加@EnableScheduling

```java
@SpringBootApplication
@EnableTransactionManagement // 开启事务
@ComponentScan("com.smart") // 扫描service和controller
@MapperScan("com.smart.mapper") // 扫描mapper
@EnableAspectJAutoProxy//允许动态代理
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

 - 定义任务

```java
@Service
@Slf4j
public class ScheduledTiming {
	@Scheduled(fixedRate = 1000)
	public void myTask() throws Exception {
		log.info("...........定时任务开始执行........");
	}
}
```
上面的定时任务，只需要在方法上添加@Scheduled注解，就这么简单，就实现了定时执行，任务的执行周期也可以自定义，固定周期和表达式都支持，详细的可以参考Scheduled的源码
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package org.springframework.scheduling.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Repeatable;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Repeatable(Schedules.class)
public @interface Scheduled {
    String cron() default "";

    String zone() default "";

    long fixedDelay() default -1L;

    String fixedDelayString() default "";

    long fixedRate() default -1L;

    String fixedRateString() default "";

    long initialDelay() default -1L;

    String initialDelayString() default "";
}

```

> 直接把定时任务的声明和执行周期通过注解的方式实现，的确方便了很多，也省去配置，传统的实现方式往往需要在xml文件中配置task以及task的trigger，Spring Boot的思想是约定由于配置，所以很多地方都是尽量不用配置的方式。