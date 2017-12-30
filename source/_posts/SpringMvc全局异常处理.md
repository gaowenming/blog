---
title: SpringMvc全局异常处理
date: 2017-12-28 22:27:08
tags: 异常处理
categories: java基础
---

> 异常处理是开发中不可避免的，好的异常处理方式，能提高代码的可读性，使代码更简洁，本文就介绍下在开发Rest接口时，如何优雅的处理异常。
<!-- more -->

如何抛出异常
---

> java中异常分为2类：预期异常和运行时异常RuntimeException，预期异常就是需要明确的捕获异常，比如文件读写时，需要捕获IOException。运行时异常，无法预知的异常，比如著名的NullPointException。那么我们在定义接口时，怎么定义异常呢？一个原则，就是尽量在调用的最上层处理。

> 比如一个调用连中，Http->Controller->Service->Dao->DB,Controller层负责和调用端的数据交互，一般都会包含请求的响应状态，成功还是失败，返回的数据，通常我们都会定义一个通用的返回体，包含Code，Msg，ResultData，Service层负责具体的业务逻辑，Dao层负责与DB的交互。

> 正常成功的请求，可以在Controller层定义，但是各种系统的异常情况怎么优雅的处理呢？答案就是抛出异常。我们定义好业务异常(BusinessException),定义好异常code和异常信息Msg，在业务出现异常的情况时，直接抛出异常信息，由Controller层统一处理，Service层只需要在遇到异常时，直接Throw就可以，这样既能阻止程序继续往下执行，也避免了代码中无限的return。

定义业务异常
---
> 上面介绍了业务异常通用类，主要包含异常代码，异常信息，具体的定义如下：

```java
public class BusinessException extends RuntimeException {
    private static final long serialVersionUID = 1L;
    private BusinessErrorMsg businessErrorMsg;

    public BusinessException(BusinessErrorMsg businessErrorMsg) {
        this.businessErrorMsg = BusinessErrorMsg.SYSTEM_ERROR;
        this.businessErrorMsg = businessErrorMsg;
    }

    public BusinessException() {
        this.businessErrorMsg = BusinessErrorMsg.SYSTEM_ERROR;
    }

    public BusinessException(BusinessErrorMsg businessErrorMsg, Throwable cause) {
        this.businessErrorMsg = BusinessErrorMsg.SYSTEM_ERROR;
        this.businessErrorMsg = businessErrorMsg;
    }

    public BusinessException(String message, Throwable cause) {
        super(message, cause);
        this.businessErrorMsg = BusinessErrorMsg.SYSTEM_ERROR;
    }

    public BusinessException(String message) {
        super(message);
        this.businessErrorMsg = BusinessErrorMsg.SYSTEM_ERROR;
    }

    protected BusinessException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace) {
        super(message, cause, enableSuppression, writableStackTrace);
        this.businessErrorMsg = BusinessErrorMsg.SYSTEM_ERROR;
    }

    public BusinessException(Throwable cause) {
        super(cause);
        this.businessErrorMsg = BusinessErrorMsg.SYSTEM_ERROR;
    }

    public BusinessErrorMsg getBusinessErrorMsg() {
        return this.businessErrorMsg;
    }

    public void setBusinessErrorMsg(BusinessErrorMsg businessErrorMsg) {
        this.businessErrorMsg = businessErrorMsg;
    }
}

```
> 异常代码我们使用枚举的方式
```java
package com.smart.service.base;

public enum BusinessErrorMsg {
    SYSTEM_ERROR(Integer.valueOf(9999), "系统错误"),
    VALIDATION_TOKEN_NULL(Integer.valueOf(1000), "token is null"),
    VALIDATION_PARAM_ERROR(Integer.valueOf(1001), "参数错误");

    private Integer errorCode = Integer.valueOf(9999);
    private String errorMessage = "系统错误";

    private BusinessErrorMsg(Integer errorCode, String errorMessage) {
        this.errorCode = errorCode;
        this.errorMessage = errorMessage;
    }

    public Integer getErrorCode() {
        return this.errorCode;
    }

    public void setErrorCode(Integer errorCode) {
        this.errorCode = errorCode;
    }

    public String getErrorMessage() {
        return this.errorMessage;
    }

    public void setErrorMessage(String errorMessage) {
        this.errorMessage = errorMessage;
    }
}
```
使用枚举的好处就是比较直观，code和msg一一对应

全局异常处理
---
异常定义好了，那怎么能优雅的捕获呢？SpringMvc提供了全局RestControllerAdvice，顾名思义，RestController的拦截器，使用这种方式，我们就不用在Controller层使用try/catch了，直接交给异常处理器就ok了，是不是很方便呢。
```java
package com.smart.server.handler;

import com.smart.server.base.BaseJsonResult;
import com.smart.service.base.BusinessErrorMsg;
import com.smart.service.base.BusinessException;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.Arrays;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import lombok.extern.slf4j.Slf4j;

/**
 * 全局异常处理
 *
 * 2017年4月28日 下午10:14:26 <br/>
 *
 * author gaowenming version since JDK 1.8
 */
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    /**
     * 系统异常
     *
     * @author gaowenming param req param e return throws Exception since JDK 1.8
     */
    @ExceptionHandler(value = Exception.class)
    @ResponseBody
    public BaseJsonResult<Object> defaultExceptionHandler(HttpServletRequest req, Exception e) throws Exception {
        log.error("<-----------------系统响应异常----------------->", e);
        printMethodParameters(req);
        BaseJsonResult<Object> baseJsonResult = new BaseJsonResult<>();
        baseJsonResult.setStatus(HttpStatus.INTERNAL_SERVER_ERROR.value());
        baseJsonResult.setMsg("请求失败,请稍后重试！");
        return baseJsonResult;
    }

    /**
     * 业务异常
     *
     * @author gaowenming param req param e return throws Exception since JDK 1.8
     */
    @ExceptionHandler(value = BusinessException.class)
    @ResponseBody
    public BaseJsonResult<Object> smartBusinessExceptionHandler(HttpServletRequest req, BusinessException e) throws Exception {
        BaseJsonResult<Object> baseJsonResult = new BaseJsonResult<>();
        BusinessErrorMsg businessErrorMsg = e.getBusinessErrorMsg();
        log.warn("catch BusinessException,code:{},message:{}", businessErrorMsg.getErrorCode(), businessErrorMsg.getErrorMessage());
        printMethodParameters(req);
        baseJsonResult.setStatus(businessErrorMsg.getErrorCode());
        baseJsonResult.setMsg(businessErrorMsg.getErrorMessage());
        return baseJsonResult;
    }

    /**
     * 记录方法的入参
     */
    private static void printMethodParameters(HttpServletRequest request) {
        StringBuilder methodInfo = new StringBuilder();
        methodInfo.append("url=").append(request.getServletPath());
        methodInfo.append(" ;params= ");
        if (request.getQueryString() != null) {
            methodInfo.append(request.getQueryString()).append(" - ");
        } else {
            Map<String, String[]> parameters = request.getParameterMap();
            if (parameters.size() != 0) {
                methodInfo.append(" [");
            }
            for (Map.Entry<String, String[]> entry : parameters.entrySet()) {
                String key = entry.getKey();
                Object value = entry.getValue();
                String message = "";
                if (value.getClass().isArray()) {
                    Object[] args = (Object[]) value;
                    message = " " + key + "=" + Arrays.toString(args) + " ";
                } else {
                    message = key + "=" + String.valueOf(value) + " ";
                }
                methodInfo.append(message);
            }
            if (parameters.size() != 0) {
                methodInfo.append("]");
            }
        }
        log.info(methodInfo.toString());
    }
}
```
> 定义GlobalExceptionHandler，加上@RestControllerAdvice注解，这样就表示拦截Controller所有的异常情况了，我们可以使用@ExceptionHandler注解，处理不同类型的异常。推荐大家把异常发生时的入参都打印出来，这样能够方便我们在出现异常时快速的定位问题。



