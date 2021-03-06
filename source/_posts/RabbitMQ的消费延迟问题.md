---
title: RabbitMQ的消费延迟问题
date: 2018-01-27 20:47:28
tags: RabbitMQ
categories: 分布式
---

> 最近一周系统出现几次消费MQ延迟的问题，最后定位到可能和MQ的延迟队列堆积数据太多有关。
<!--more-->

异常现象
---
最近一周，线上服务出现了几次消费RabbitMQ的延迟，具体的现象就是消费端（Consumer）有20秒的时间内，没有消费任何队列中的数据，好像和MQ断开连接一样，20秒后，所有队列又能正常消费了，有几个队列因为数据有先后的顺序逻辑，造成了部分数据的异常。

问题定位
---
之前从来没有出现过这种问题，分析最近的上线内容，怀疑和上周一个同事上线的一个24小时的延迟队列有关，为什么这么说呢？这个24小时的延迟队列，大概里面是300多万的数据，占用内存是8G左右，之前就出现过消息生产端(Provider)往RabbitMQ扔消息延迟的情况。

看了下RabbitMQ的相关文档，发现好像是RabbitMQ内部有个限流机制，当数据超过了服务设置的警戒值，就会触发RabbitMQ的限流机制，比如降低消息的生产速度，这样在保证消费速度的前提下，MQ中的数据量会越来越少，但是为什么会引起Consumer无法消费的问题呢？看了下MQ的监控，发现在出现问题的时间点，MQ的波动很大，应该是出现了服务的短时间阻塞
![](/images/mq.png "")

从上面的监控图可以看出，MQ中的数据堆积到一定的数量后，会引起MQ服务的不稳定，从服务的波动上就可以看出，26日后面的曲线相对来说是比较平滑的，没有出现很剧烈的波动，因为25日就把该延迟队列迁移到另一个服务中。

问题总结
---
> 对于MQ的使用，在系统解耦方面是很常见的，延迟队列又能很好的解决我们系统上一些业务场景，比如延迟关闭过期订单。但是使用上我们还是需要注意几个问题：

 1. 是否使用ACK机制
 ACK，就是消息消费的确认机制，在消费队列中的消息时，我们可以指定是否采用ack，如果采用ack，则队列的消息需要在消费逻辑完成后通知队列，remove该条消息，如果不采用ack，默认就是consumer消费消息后，该消息自动从队列中remove。
可以根据队列自身相关业务，决定是否需要采用ack机制，对于那些比较重要的消息，我觉得还是采用ack的机制比较好。

 2. 延迟队列的数据堆积
    延迟队列，意味着数据延迟消费，当然如果延迟时间比较长，那么堆积的数据也会越来越多，如果队列比较多的话，推荐大家把占用内存比较大，数据量比较多的延迟队列单独出来，这样即使出现性能问题，也不会影响到别的队列的正常运行。

 


