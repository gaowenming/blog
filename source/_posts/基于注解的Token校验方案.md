---
title: 基于注解的Token校验方案
date: 2017-12-26 21:08:29
tags: 注解
categories: java基础
---


> 在后端开发中，通常需要对接口做权限校验，校验用户是否需要登录，是否需要认证等等，本文就来介绍下如何通过注解的方式来对Token做认证
<!-- more -->

注解定义
---

> 其实说到用户认证，大家脑海中肯定会想到拦截器（Interceptor）和过滤器（Filter），那么怎样才能更灵活的实现呢？注解毫无疑问就派上用场了。
 
首先定义注解：
```java
package com.smart.server.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * token注解，可用于类、方法中，拦截token验证
 *
 * Created by gaowenming on 2017/6/15.
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface TokenValidation {
    String name() default "";
}

```
定义一个注解TokenValidation，Target的作用范围是TYPE和METHOD，也就是可以在Class类上，也可以在方法上，如果是在类上使用注解，表示该类所有的方法都需要进行Token的校验，如果在方法上，表示该方法需要进行认证。

注解实现
---
>注解定义好了,我们基于AOP的方式，对注解进行拦截
```java

import com.smart.server.util.Constants;
import com.smart.service.base.BusinessErrorMsg;
import com.smart.service.base.BusinessException;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.context.request.RequestAttributes;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;

import lombok.extern.slf4j.Slf4j;

/**
 * Created by gaowenming on 2017/6/15.
 */
@Aspect
@Component
@Slf4j
public class TokenValidationInterceptor {

    //注解在类上面@within
    @Pointcut("@within(com.smart.server.annotation.TokenValidation)")
    public void pointcut() {
    }

    @Before("pointcut()")
    public void tokenValidationType(JoinPoint point) throws Throwable {
        commonTokenValidation(point);
    }


    //注解在方法上面@annotation
    @Pointcut("@annotation(com.smart.server.annotation.TokenValidation)")
    public void pointcutMethod() {
    }

    @Before("pointcutMethod()")
    public void tokenValidationMethod(JoinPoint point) throws Throwable {
        commonTokenValidation(point);
    }


    //公共的校验
    public static void commonTokenValidation(JoinPoint point) throws Throwable {
        HttpServletRequest request;

        RequestAttributes ra = RequestContextHolder.getRequestAttributes();
        ServletRequestAttributes sra = (ServletRequestAttributes) ra;
        request = sra.getRequest();

        String url = request.getServletPath();
        String token = request.getHeader(Constants.TOKEN_NAME);
        log.info("TokenHandlerInterceptor----- url:{},token:{}  ", url, token);
        if (StringUtils.isEmpty(token)) {
            throw new BusinessException(BusinessErrorMsg.VALIDATION_TOKEN_NULL);
        }

        //校验token是否过期和正确
        //TODO
    }

}

```
上面的实现，做个简单的说明，首先我们约定好，token封装在Http请求的Header中，拦截器上中先从Header中获取token的值，然后判断token是否为空，如果不为空，还需要判断token是否过期，是否正确。在实际项目中，用户登录后，服务端会根据一定的算法计算出token的值，然后把该token的值放入Redis中，并设置有效期，这样就很方便的处理过期的问题。

注解使用
---
经过上面的2步，就可以在我们定义的Controller中使用我们定义的注解了
```java
package com.smart.server.controller;

import com.smart.model.Dic;
import com.smart.server.annotation.TokenValidation;
import com.smart.server.base.BaseController;
import com.smart.server.base.BaseJsonResult;
import com.smart.service.IDicService;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import io.swagger.annotations.Api;
import lombok.extern.slf4j.Slf4j;


@RestController
@RequestMapping("api/dic/")
@Api
@Slf4j
public class DicController extends BaseController {

    @Autowired
    private IDicService dicService;

    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public BaseJsonResult<Dic> getDic(@PathVariable Integer id) throws Exception {
        log.info("getDic.......");
        Dic dic = dicService.get(id);
        return successResult(dic);
    }

    @RequestMapping(value = "/addDic", method = RequestMethod.POST)
    @TokenValidation
    public BaseJsonResult addDic(@RequestBody Dic dic) throws Exception {
        log.info("addDic.......");
        dicService.save(dic);
        return successNullDataResult();
    }
}
```

是不是很方便，get操作不需要校验，就无需增加注解，在add操作中，增加token校验，加上注解即可。

为什么用注解
---
>使用注解的方式，你会发现非常灵活，类和方法可以自己灵活设置，处理逻辑使用AOP的方式统一处理。其实还有其他的一些使用场景，比如有些接口返回值需要加密返回，也可以使用类似的注解方式实现。