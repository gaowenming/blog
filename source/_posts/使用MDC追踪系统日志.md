---
title: 使用MDC追踪系统日志
date: 2017-12-30 09:48:16
tags: MDC
categories: 分布式
---





> 在系统开发中，日志必不可少，前段时间我们还因为在线上开启Debug而造成系统故障，所以日志的处理应该让我们更加重视。在线上系统，日志很多，接口之间的日志错乱交替的打印，大大的影响了我们对日志的跟踪，我们要想跟踪一次请求的完整日志，好像都只能通过时间、线程名等信息，但是也无法精确的找出。MDC的出现，完美的解决了这个棘手的问题。
<!-- more -->

什么是MDC
---

> MDC ( Mapped Diagnostic Contexts )，是SLF4J中提供的一个类，专门用来跟踪请求链路的日志
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package org.slf4j;

import java.io.Closeable;
import java.util.Map;
import org.slf4j.helpers.NOPMDCAdapter;
import org.slf4j.helpers.Util;
import org.slf4j.impl.StaticMDCBinder;
import org.slf4j.spi.MDCAdapter;

public class MDC {
    static final String NULL_MDCA_URL = "http://www.slf4j.org/codes.html#null_MDCA";
    static final String NO_STATIC_MDC_BINDER_URL = "http://www.slf4j.org/codes.html#no_static_mdc_binder";
    static MDCAdapter mdcAdapter;

    private MDC() {
    }

    private static MDCAdapter bwCompatibleGetMDCAdapterFromBinder() throws NoClassDefFoundError {
        try {
            return StaticMDCBinder.getSingleton().getMDCA();
        } catch (NoSuchMethodError var1) {
            return StaticMDCBinder.SINGLETON.getMDCA();
        }
    }

    public static void put(String key, String val) throws IllegalArgumentException {
        if (key == null) {
            throw new IllegalArgumentException("key parameter cannot be null");
        } else if (mdcAdapter == null) {
            throw new IllegalStateException("MDCAdapter cannot be null. See also http://www.slf4j.org/codes.html#null_MDCA");
        } else {
            mdcAdapter.put(key, val);
        }
    }

    public static MDC.MDCCloseable putCloseable(String key, String val) throws IllegalArgumentException {
        put(key, val);
        return new MDC.MDCCloseable(key);
    }

    public static String get(String key) throws IllegalArgumentException {
        if (key == null) {
            throw new IllegalArgumentException("key parameter cannot be null");
        } else if (mdcAdapter == null) {
            throw new IllegalStateException("MDCAdapter cannot be null. See also http://www.slf4j.org/codes.html#null_MDCA");
        } else {
            return mdcAdapter.get(key);
        }
    }

    public static void remove(String key) throws IllegalArgumentException {
        if (key == null) {
            throw new IllegalArgumentException("key parameter cannot be null");
        } else if (mdcAdapter == null) {
            throw new IllegalStateException("MDCAdapter cannot be null. See also http://www.slf4j.org/codes.html#null_MDCA");
        } else {
            mdcAdapter.remove(key);
        }
    }

    public static void clear() {
        if (mdcAdapter == null) {
            throw new IllegalStateException("MDCAdapter cannot be null. See also http://www.slf4j.org/codes.html#null_MDCA");
        } else {
            mdcAdapter.clear();
        }
    }

    public static Map<String, String> getCopyOfContextMap() {
        if (mdcAdapter == null) {
            throw new IllegalStateException("MDCAdapter cannot be null. See also http://www.slf4j.org/codes.html#null_MDCA");
        } else {
            return mdcAdapter.getCopyOfContextMap();
        }
    }

    public static void setContextMap(Map<String, String> contextMap) {
        if (mdcAdapter == null) {
            throw new IllegalStateException("MDCAdapter cannot be null. See also http://www.slf4j.org/codes.html#null_MDCA");
        } else {
            mdcAdapter.setContextMap(contextMap);
        }
    }

    public static MDCAdapter getMDCAdapter() {
        return mdcAdapter;
    }

    static {
        try {
            mdcAdapter = bwCompatibleGetMDCAdapterFromBinder();
        } catch (NoClassDefFoundError var2) {
            mdcAdapter = new NOPMDCAdapter();
            String msg = var2.getMessage();
            if (msg == null || !msg.contains("StaticMDCBinder")) {
                throw var2;
            }

            Util.report("Failed to load class \"org.slf4j.impl.StaticMDCBinder\".");
            Util.report("Defaulting to no-operation MDCAdapter implementation.");
            Util.report("See http://www.slf4j.org/codes.html#no_static_mdc_binder for further details.");
        } catch (Exception var3) {
            Util.report("MDC binding unsuccessful.", var3);
        }

    }

    public static class MDCCloseable implements Closeable {
        private final String key;

        private MDCCloseable(String key) {
            this.key = key;
        }

        public void close() {
            MDC.remove(this.key);
        }
    }
}

```
> MDC中提供的全是静态方法，实现的核心是**MDCAdapter**，从命名方式，我们不难理解，这是适配器，这也是SLF4J的核心，提供适配器，具体的实现有Log4j或者LogBack。
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package org.slf4j.spi;

import java.util.Map;

public interface MDCAdapter {
    void put(String var1, String var2);

    String get(String var1);

    void remove(String var1);

    void clear();

    Map<String, String> getCopyOfContextMap();

    void setContextMap(Map<String, String> var1);
}

```
> 我们使用的Logback，其中对MDCAdapter接口有具体的实现类**LogbackMDCAdapter**

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package ch.qos.logback.classic.util;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;
import org.slf4j.spi.MDCAdapter;

public class LogbackMDCAdapter implements MDCAdapter {
    final ThreadLocal<Map<String, String>> copyOnThreadLocal = new ThreadLocal();
    private static final int WRITE_OPERATION = 1;
    private static final int MAP_COPY_OPERATION = 2;
    final ThreadLocal<Integer> lastOperation = new ThreadLocal();

    public LogbackMDCAdapter() {
    }

    private Integer getAndSetLastOperation(int op) {
        Integer lastOp = (Integer)this.lastOperation.get();
        this.lastOperation.set(op);
        return lastOp;
    }

    private boolean wasLastOpReadOrNull(Integer lastOp) {
        return lastOp == null || lastOp.intValue() == 2;
    }

    private Map<String, String> duplicateAndInsertNewMap(Map<String, String> oldMap) {
        Map<String, String> newMap = Collections.synchronizedMap(new HashMap());
        if (oldMap != null) {
            synchronized(oldMap) {
                newMap.putAll(oldMap);
            }
        }

        this.copyOnThreadLocal.set(newMap);
        return newMap;
    }

    public void put(String key, String val) throws IllegalArgumentException {
        if (key == null) {
            throw new IllegalArgumentException("key cannot be null");
        } else {
            Map<String, String> oldMap = (Map)this.copyOnThreadLocal.get();
            Integer lastOp = this.getAndSetLastOperation(1);
            if (!this.wasLastOpReadOrNull(lastOp) && oldMap != null) {
                oldMap.put(key, val);
            } else {
                Map<String, String> newMap = this.duplicateAndInsertNewMap(oldMap);
                newMap.put(key, val);
            }

        }
    }

    public void remove(String key) {
        if (key != null) {
            Map<String, String> oldMap = (Map)this.copyOnThreadLocal.get();
            if (oldMap != null) {
                Integer lastOp = this.getAndSetLastOperation(1);
                if (this.wasLastOpReadOrNull(lastOp)) {
                    Map<String, String> newMap = this.duplicateAndInsertNewMap(oldMap);
                    newMap.remove(key);
                } else {
                    oldMap.remove(key);
                }

            }
        }
    }

    public void clear() {
        this.lastOperation.set(Integer.valueOf(1));
        this.copyOnThreadLocal.remove();
    }

    public String get(String key) {
        Map<String, String> map = (Map)this.copyOnThreadLocal.get();
        return map != null && key != null ? (String)map.get(key) : null;
    }

    public Map<String, String> getPropertyMap() {
        this.lastOperation.set(Integer.valueOf(2));
        return (Map)this.copyOnThreadLocal.get();
    }

    public Set<String> getKeys() {
        Map<String, String> map = this.getPropertyMap();
        return map != null ? map.keySet() : null;
    }

    public Map<String, String> getCopyOfContextMap() {
        Map<String, String> hashMap = (Map)this.copyOnThreadLocal.get();
        return hashMap == null ? null : new HashMap(hashMap);
    }

    public void setContextMap(Map<String, String> contextMap) {
        this.lastOperation.set(Integer.valueOf(1));
        Map<String, String> newMap = Collections.synchronizedMap(new HashMap());
        newMap.putAll(contextMap);
        this.copyOnThreadLocal.set(newMap);
    }
}

```

> 从源码中看出，核心思想还是ThreadLocal
```java
    final ThreadLocal<Map<String, String>> copyOnThreadLocal = new ThreadLocal();
    private static final int WRITE_OPERATION = 1;
    private static final int MAP_COPY_OPERATION = 2;
    final ThreadLocal<Integer> lastOperation = new ThreadLocal();
```
定义了2个ThreadLocal，一个维护着Map<String, String>，这就是我们在使用时自己定义的追踪数据，另一个ThreadLocal记录着该线程的操作记录，需要注意，在上面的代码中，write操作即put会去修改lastOperation，而get操作则不会。这样就保证了，只会在第一次写时复制。

MDC的使用
---

定义全局拦截器
===============

>  MDC的使用很简单，只需要调用put方法就行，MDC.put("traceId", "traceId"),再结合HandlerInterceptor的使用，可以在所有的请求中都加上日志追踪，就无需手动在每个请求中加了。
```java
package com.smart.server.interceptor;

import org.slf4j.MDC;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import java.lang.reflect.Method;
import java.util.UUID;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import lombok.extern.slf4j.Slf4j;

/**
 * ClassName: TimeHandlerInterceptor <br/> Function: 方法执行时间拦截器. <br/> date: 2017年3月23日 下午8:46:58
 * <br/>
 *
 * author gaowenming since JDK 1.8
 */
@Slf4j
public class TimeHandlerInterceptor implements HandlerInterceptor {

    // 当前时间戳
    private ThreadLocal<Long> threadLocalTime = new ThreadLocal<>();

    /**
     * controller 执行之前调用
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //日志追加traceId追踪
        MDC.put("traceId", "traceId=" + UUID.randomUUID().toString().replace("-", ""));
        long startTime = System.currentTimeMillis();
        threadLocalTime.set(startTime);
        return true;
    }

    /**
     * controller 执行之后，且页面渲染之前调用
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        HandlerMethod handlerMethod = (HandlerMethod) handler;
        Method method = handlerMethod.getMethod();
        long endTime = System.currentTimeMillis();
        long startTime = threadLocalTime.get();
        long executeTime = endTime - startTime;
        log.info("[{}.{}] 执行耗时:{}ms", method.getDeclaringClass().getName(), method.getName(), executeTime);
    }

    /**
     * 页面渲染之后调用，一般用于资源清理操作
     */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        MDC.clear();
    }

}
```

定义一个全局拦截器，拦截所有的Controller请求，在preHandle方法中，调用MDC.put("traceId", "traceId=" + UUID.randomUUID().toString().replace("-", ""));这样traceId就会被加到该请求所有的链路中，在afterCompletion方法中clear，确保本次请求完成后，MDC中的数据要清空。
该拦截器中还加入了方法的执行时间统计，原理和MDC类似，使用ThreadLocal。

修改logback.xml文件
===============

> 上面我们往MDC中put了traceId的key，再把这个key增加到logback.xml文件中即可
```xml
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%date %level %X{traceId} [%thread] %logger{50}-[%line] %msg%n</pattern>
        </encoder>
    </appender>
```
配置完logback文件后，然后就可以正常打印了

```java
2017-12-30 10:48:53,986 INFO traceId=73eb01785cce4ce8812dd091b07c2ccd [http-nio-8000-exec-8] 
2017-12-30 10:48:53,987 WARN traceId=73eb01785cce4ce8812dd091b07c2ccd [http-nio-8000-exec-8] 
2017-12-30 10:48:53,988 ERROR traceId=73eb01785cce4ce8812dd091b07c2ccd [http-nio-8000-exec-8] 
```

到此，MDC的使用就介绍完了，其实总结起来也挺简单的，核心就是ThreadLocal，配置也非常简单，但是功能确很强大，很大的方便了我们对日志的追踪。