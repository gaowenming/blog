---
title: Redis解决定时任务在分布式环境单节点执行
date: 2017-12-24 19:02:21
tags: Redis
categories: 分布式
---


> 定时任务，在日常开发中是经常用到的，在分布式环境中，定时任务的执行往往需要控制多节点同时执行的问题，比如可以借助Quartz，或者分布式锁，本文提供另一种解决方式，也是借助Redis。
<!-- more -->

实现方式
---


 >在Redis中，提供了自增和增减的相关命令，来保证计数的原子性
 
**incr** 递增1并返回递增后的结果；
**incrby** 根据指定值做递增或递减操作并返回递增或递减后的结果(incrby递增或递减取决于传入值的正负)
**decr** 递减1并返回递减后的结果；
**decrby** 根据指定值做递增或递减操作并返回递增或递减后的结果(decrby递增或递减取决于传入值的正负)

了解了上面几个命令之后，在来说通过计数器来控制并发执行，应该就很容易理解了
直接上代码吧：
```java
    long jobKeyValue = redisKeyValueResolver.increment(ExdataConstants.JOB_KEY);
    LOGGER.debug("jobKeyIncrementAfterValue:{}", jobKeyValue);
        if (jobKeyValue == 1) {
            //一秒失效
            redisKeyValueResolver.expireKey(ExdataConstants.JOB_KEY, 1);
            //业务逻辑。。。
        }
```
上面一段代码，首先定义一个key，当定时任务触发时，多个节点同时执行incr自增操作,第一个执行自增的节点的值就是1，其他的节点就是2、3、4...根据自增后的值来控制定时任务的执行，也就可以有效的控制只执行一次了。

>当然还是需要注意，一定要在if语句块最前面执行expire操作，因为执行业务逻辑需要时间，更有可能出现异常，所以在最前面设置过期时间，保证了下次定时任务执行时，key已失效。

不需要使用较重的分布式锁，巧妙的运用了Redis的原子操作，这样是的实现方式是不是更简单呢?
