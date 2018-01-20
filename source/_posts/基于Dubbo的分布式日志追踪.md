---
title: 基于Dubbo的分布式日志追踪
date: 2017-12-30 19:11:55
tags: Dubbo
categories: 分布式
---




> 之前讲过单系统中的日志追踪，现在系统设计都是往服务化的方向发展，一个大型项目被拆分成多个服务，服务之间互相调用。拆分的服务越多，服务之间的关系越复杂，日志的追踪就更加困难。那么RPC之间怎么做到日志追踪呢，本文就介绍下基于Dubbo的日志追踪。
<!-- more -->
Dubbo的Filter接口
---
> 我们需要先介绍下Dubbo的Filter接口
```java
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.SPI;

@SPI
public interface Filter {
    Result invoke(Invoker<?> var1, Invocation var2) throws RpcException;
}

```
接口的定义很简单，只有一个invoke方法，由此就可以得到我们的实现思路，在Dubbo的调用端生成traceId，在调用的时候把该traceId传递到服务端，服务端拿到这个traceId，用该traceId作为本次调用的追踪id，Dubbo也提供了RpcContext类，用于Consumer到Provider的参数传递
```java
public class RpcContext {
    private static final ThreadLocal<RpcContext> LOCAL = new ThreadLocal<RpcContext>() {
        protected RpcContext initialValue() {
            return new RpcContext();
        }
    };
    private Future<?> future;
    private List<URL> urls;
    private URL url;
    private String methodName;
    private Class<?>[] parameterTypes;
    private Object[] arguments;
    private InetSocketAddress localAddress;
    private InetSocketAddress remoteAddress;
    private final Map<String, String> attachments = new HashMap();
    private final Map<String, Object> values = new HashMap();
```
RpcContext中有attachments属性，参数可以通过key-value的形式传递

Consumer端生成traceId
---
```java
public class TraceFilter implements Filter {

    private static Logger LOGGER = LoggerFactory.getLogger(TraceFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        RpcContext context = RpcContext.getContext();
        String chTraceId = MDC.get("traceID");
        LOGGER.debug("chTraceId={}", chTraceId);
        context.setAttachment("chTraceId", chTraceId);
        return invoker.invoke(invocation);
    }
}

```
调用端发起请求前，往RpcContext中加入我们自己定义的traceId

Provider端接收traceId
---
```java
public class TraceFilter implements Filter {

    private static Logger LOGGER = LoggerFactory.getLogger(TraceFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        RpcContext context = RpcContext.getContext();
        String chTraceId = context.getAttachment("chTraceId") + "&client=" + context.getRemoteAddressString();
        LOGGER.debug("chTraceId={}", chTraceId);
        if (StringUtils.isNotEmpty(chTraceId)) {
            MDC.put("traceID", chTraceId);
        }
        Result result = invoker.invoke(invocation);
        if (StringUtils.isNotEmpty(chTraceId)) {
            MDC.clear();
        }
        return result;
    }
}
```
服务提供端收到traceId，然后使用该id做为后续整个接口链路的日志追踪id
实现其实很简单，只需要分别在服务的2端实现Filter接口，使用RpcContext发送和接收参数
最后还需在resource目录下定义自己的Filter，新建META-INF\dubbo目录，定义一个文件，文件名：
com.alibaba.dubbo.rpc.Filter，里面增加一行自己定义的Filter，内容如下：traceFilter=com.wlqq.ms.filter.TraceFilter
详细的配置可以参考Dubbo的官方说明文档：http://dubbo.io/books/dubbo-user-book/demos/attachment.html 

总结
---
分布式链路追踪其实是一个复杂的项目，本文只是一个简单的实现，Google提出了dapper论文，Twitter开源的Zipkin，后面会介绍下Zipkin的使用，也打算在项目中引入Zipkin，毕竟现在这种实现还是无法满足复杂的场景。
