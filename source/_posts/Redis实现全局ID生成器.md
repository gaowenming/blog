---
title: Redis实现全局ID生成器
date: 2018-05-09 09:42:00
tags: Redis
categories: 分布式
---



> 在分布式系统架构中，经常都需要一个全局的ID生成器，来保证系统中某些业务场景中对于主键的要求，当前实现ID生成的方式还是挺多的，本文我们主要介绍下如何使用Redis的incr机制实现全局ID生成。
<!--more-->

Redis的incr命令
---

对存储在指定key的数值执行原子的加1操作。

如果指定的key不存在，那么在执行incr操作之前，会先将它的值设定为0。

如果指定的key中存储的值不是字符串类型或者存储的字符串类型不能表示为一个整数

那么执行这个命令时服务器会返回一个错误(eq:(error) ERR value is not an integer or out of range)。


实现思路
---
> 1、定义一个通用的key，该key会的规则是时间格式，精确到秒，保证每秒都是不同的key，value的值是一个long型的整数，前半部分是当前时间精确到秒，后面是自增的值，设计成5位，不够的补0，这样基本就是每秒最多能生成99999个ID，基本能满足大部分的需求，如果需要更多，可以多保留几位就行。

```java
/**
 * 使用redis生成分布式ID
 */
public interface IdGeneratorService {

    /**
     * @param biz 业务名称
     */
    long generatorId(String biz);

    /**
     *
     * @return
     */
    long generatorId();

```

```java
package com.smart.server.idgen.redis;

import com.google.common.base.Strings;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Service;

import java.util.Calendar;
import java.util.Date;
import java.util.concurrent.TimeUnit;

import lombok.extern.slf4j.Slf4j;

@Service
@Slf4j
public class RedisIdGeneratorService implements IdGeneratorService {

    private static final String keyPrefix = "smart";

    /**
     * JedisClient对象
     */
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;


    /**
     * @Description
     * @author butterfly
     */
    private String getIDPrefix() {
        Date date = new Date();
        Calendar c = Calendar.getInstance();
        c.setTime(date);
        int year = c.get(Calendar.YEAR);
        int day = c.get(Calendar.DAY_OF_YEAR); // 今天是第多少天
        int hour = c.get(Calendar.HOUR_OF_DAY);
        int minute = c.get(Calendar.MINUTE);
        int second = c.get(Calendar.SECOND);
        String dayFmt = String.format("%1$03d", day); // 0补位操作 必须满足三位
        String hourFmt = String.format("%1$02d", hour);  // 0补位操作 必须满足2位
        String minuteFmt = String.format("%1$02d", minute);  // 0补位操作 必须满足2位
        String secondFmt = String.format("%1$02d", second);  // 0补位操作 必须满足2位
        StringBuffer prefix = new StringBuffer();
        prefix.append((year - 2000)).append(dayFmt).append(hourFmt).append(minuteFmt).append(secondFmt);
        return prefix.toString();
    }


    /**
     * @author butterfly
     */
    private long incrDistrId(String biz) {
        String prefix = getIDPrefix();
        String orderId = null;
        String key = "#{biz}:id:".replace("#{biz}", biz).concat(prefix); // 00001
        try {
            ValueOperations<String, Object> valueOper = redisTemplate.opsForValue();
            Long index = valueOper.increment(key, 1);
            orderId = prefix.concat(String.format("%1$05d", index)); // 补位操作 保证满足5位
        } catch (Exception ex) {
            log.error("分布式订单号生成失败异常。。。。。", ex);
        } finally {
            redisTemplate.expire(key, 600, TimeUnit.SECONDS);//保留10分钟内的key
        }
        if (Strings.isNullOrEmpty(orderId)) return 0;
        return Long.parseLong(orderId);
    }

    /**
     * @Description 生成分布式ID
     * @author butterfly
     */
    @Override
    public long generatorId(String biz) {
        // 转成数字类型，可排序
        return incrDistrId(biz);
    }

    @Override
    public long generatorId() {
        return incrDistrId(keyPrefix);
    }
}

```

单元测试
---
```java
package com.test;

import com.smart.server.Application;
import com.smart.server.idgen.redis.IdGeneratorService;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import lombok.extern.slf4j.Slf4j;

/**
 * @author gaowenming
 * @create 2018-05-08 11:10
 * @desc
 **/
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
@Slf4j
public class IDTest {

    @Autowired
    private IdGeneratorService idGeneratorService;

    @Test
    public void testRedisId() throws InterruptedException {
        for (int i = 0; i < 1000; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    log.info(idGeneratorService.generatorId() + "");
                }
            }).start();
        }

        Thread.sleep(10000);
    }
}

```

 
总结
---
> Redis实现ID生成的方式实现起来还是挺简单的，当然优缺点也是存在的
优点：
1、能够一定程度上保持Id具有一定的增长规律，有利于索引和排序
2、比较灵活，可以根据自己的需要，定制不同的ID的格式
缺点：
1、依赖于Redis，需要引入Redis中间件的配置
2、服务端统一生成，和本地生成的效率相比，还是差一点
 


