---
title: 聊聊Dubbo接口授权
date: 2018-01-19 11:40:39
tags: Dubbo
categories: 分布式
---


> 上一篇博客讲了如何使用Dubbo的Filter实现日志追踪，其实Filter的功能有很多，今天讲讲如何使用Filter实现Dubbo的接口授权
<!-- more -->

Dubbo本身是没有授权机制的，所以需要我们自己实现，具体的实现方案如下：
![实现流程](/images/dubbo.jpg "实现流程图")

授权的粒度，主要有如下几种：
1、全部接口
2、部分接口
3、单个接口

授权服务本身的稳定性也是十分重要的，不要因为授权服务的不可用造成正常业务的瘫痪。
伪代码如下：
```java
public class AuthFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        if (success) {
            return invoker.invoke(invocation);
        }
        else {
            // TODO 错误处理
        }
    }
}
```
原理也很简单，在执行invoke之前先对consumer的请求做校验，首先dubbo调用时，公共参数都是可以获取的，比如方法名称，consumer端的名称等，当然还可以通过下发appId的方式，来区分调用端
```java
public class ConsumerContextFilter implements Filter {
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        // 设置appId
        RpcContext.getContext().setAttachment("appId", appId);
        // 执行接口
        return invoker.invoke(invocation);
    }
}
```
consumer端把自身的appid传到provider，provider会根据appid来区分不同调用的权限

> Dubbo的Filter用处确实很多，而且使用也很方便，通过2篇文章讲解了日志和权限的应用，能够在不侵入业务的情况下完成既定的功能，非常灵活。