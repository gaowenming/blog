---
title: Spring Boot中Properties文件的解析
date: 2018-01-21 11:20:21
tags: Properties
categories: SpringBoot
---

> 项目中经常会把一些公共的配置放在Properties文件中，然后通过一些方法读取Properties文件的属性名和对应的值，通常都是解析成Map<K,V>格式，在Spring Boot中，还提供了另一种更灵活的方式：自定义类，和Properties文件一一对应。
<!-- more -->

新建Properties文件
---
新建一个文件，存放一些公共的配置
```properties
#1、组件集成之外的自定义配置，最好和部署环境无关的，和环境相关的，还是配置在application文件中
#2、所有的key都加上smart前缀,防止和application中的key冲突
#3、当前key/value的解析在SmartConfigProperties文件中，新增的key需要再类中声明即可
smart.username=gaowenming
smart.secretKey=123456789
smart.id=1000
smart.names=zhangsan,lisi
smart.weight=60.5
```
在上面的文件中，我们都加上smart前缀，这样更好的避免和别的配置重名

定义配置的公共类
---
```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;

import lombok.Data;

/**
 * 业务配置 Author: gaowenming Description: Date: Created in 11:59 2017/7/15.
 */
@Configuration
@PropertySource("classpath:config.properties")//注意路径
@ConfigurationProperties(prefix = "smart")
@Data
public class SmartConfigProperties {
    private String username;
    private String secretKey;
    private int id;
    private String[] names;
    private float weight;
}
```
Spring Boot中，可以支持Properties文件到类的类型转换，常规的类型和数组，List都支持，这样避免了我们自己解析后再强制类型转换，实现方式更优雅。

使用配置
---
这种方式很好的把所有的配置都集中到一起，如果配置太多，我们也可以分多个类，避免一个类中太臃肿。使用起来也是很方便，直接把配置类注入到使用的地方即可
```java
@RestController
@RequestMapping("api/test/")
@Api
@Slf4j
public class TestController extends BaseController {

    @Resource
    private SmartConfigProperties smartConfigProperties;

    @RequestMapping(value = "getConfigValue", method = RequestMethod.GET)
    public BaseJsonResult getConfigValue() throws Exception {
        String[] names = smartConfigProperties.getNames();
        for (String str : names) {
            log.info(str);
        }
        int id = smartConfigProperties.getId();
        String username = smartConfigProperties.getUsername();
        float weight = smartConfigProperties.getWeight();
        log.info("names:{},id:{},username:{},weight:{}", names, id, username, weight);
        return successNullDataResult();
    }
}
```

> 使用这种方式解析Properties文件,首先不需要我们自己在强制转换数据类型，也使配置都集中管理，确实是给我们的代码带来更多的可读性，虽然功能简单，但是作用却很大。



