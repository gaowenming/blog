---
title: Elasticsearch(五) java客户端介绍
date: 2018-02-06 17:40:20
tags: Elasticsearch
categories: Elasticsearch
---


> 操作Elasticsearch的api众多，java语言领域主要有3种，和官方server一起发布的Transport client，searchbox-io提供的Jest，还有Spring-Data提供的Spring Data ElasticSearch
<!--more-->

Transport客户端
---
> java transport api提供了Query Builder来协助构建查询对象，而http则需要自己在代码里拼JSON DSL，从程序员角度来说， java transport api更显得更加友好，并且性能也要比http稍好。
> 
但java transport api也有如下弊病:
1. 第三方依赖包比较多，如果应用还要集成其他一些框架和组件，容易产生依赖冲突，解决起来比较麻烦。 
2. client版本必须和ES服务端版本一致，否则容易产生兼容性问题。 
3. client端JAVA版本也需要和Server端保持一致，否则也可能产生兼容性问题。
4. client端的环境和版本需要和server端保持一致这个要求，使得client/server端运行环境强耦合，导致ES Server端很难独立升级。
 
**官方的roadmap也指明，未来java transport api会被取消，建议使用rest client。**

Spring Data ElasticSearch
---
> Spring Data ElasticSearch 也是提供了一层封装，使得操作Elasticsearch变得更简单，但是扩展性不强，而且最近更新的比较慢，使得后面的最新版本都无法支持

Searchbox-io jest
---
jest是Searchbox提供的基于Http协议的客户端
> jest是一款java Rest client，它支持SSL、proxy等等，而原生的transport client是TCP连接，用户认证授权要自己想办法实现，并且封装接口控制用户的操作。

**原生的API没有任何安全保护层，而对于HTTP来说加一层安全认证是比较简单的。**

jest client和原生的client的比较还有：
> 
如果ES集群的版本不同，用HTTP client会好些。如果版本不是问题，原生的client可能更好。因为它是culster-aware的，并且可以从ES集群分担一部分计算，比如合并搜索结果是在本地client执行的而不是data node。
> 
原生的client像一个单纯的节点连接集群，它知道集群的状态和路由请求，这样会消耗多一些内存，在生产环境是值得考虑的影响。

如果使用SpringBoot的话，已经内置了starter，很快速的就能接入
```xml
#jest
spring.elasticsearch.jest.uris=http://localhost:9200
spring.elasticsearch.jest.read-timeout=10000
spring.elasticsearch.jest.username=elastic
spring.elasticsearch.jest.password=changme
```

总结
---
> 1、首先对于Elasticsearch的版本敏感度来说，jest几乎是不怎么受影响的，但是Transport和SpringData要求版本一致性更高
> 2、client的更新速度上，transport和jest都还不错，能最快速度支持新版本，SpringData最差，现在已经跟不上版本的发布速度了
> 3、Transport计划从发布中废除，而且安全上，也是不利于维护

从以上几点比较上，当前阶段，最推荐的方式还是使用jest来操作Elasticsearch。

