---
title: Elasticsearch(四) IK分词器
date: 2018-02-06 17:39:28
tags: Elasticsearch
categories: Elasticsearch
---


> 分词是Elasticsearch中非常重要的组件，毕竟官方的分词器很傻瓜，无法满足实际的业务需求。
<!--more-->

IK分词器
---
> 这里主要介绍下IK分词器的安装和使用方法

 **1. 安装IK插件**
> 安装IK分词插件非常简单， github地址：https://github.com/medcl/elasticsearch-analysis-ik/ 
需要注意的是，ELasticsearch是版本非常敏感的，所以在安装IK插件的时候，一定要注意和自己的版本对应。
我使用的版本是5.6.0
```bash
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.6.0/elasticsearch-analysis-ik-5.6.0.zip
```

安装完出现
```bash
Installed analysis-ik 
```
表示安装成功。
测试一下：
![](/images/ik.png)


 2. mapping中指定IK分词方式
 如果某个字段需要进行分词，那么在设计mapping时，就需要指定。
```java
 .startObject("properties")
                    .startObject("id").field("type", "long").field("store", "yes").endObject()
                    //设置分词
                    .startObject("title").field("type", "string").field("analyzer", "ik_max_word").field("store", "no").endObject()
                    .startObject("content").field("type", "string").field("analyzer", "ik_max_word").field("store", "yes").endObject()
                    //不分词
                    .startObject("author").field("type", "string").field("index", "not_analyzed").field("store", "no").endObject()
                    .startObject("tags").field("type", "string").field("index", "not_analyzed").field("store", "no").endObject()
                    .startObject("categroy").field("type", "string").field("index", "not_analyzed").field("store", "no").endObject()
                    .startObject("createTime").field("type", "long").field("index", "not_analyzed").field("store", "no").endObject()
                    .startObject("lastUpdateTime").field("type", "date").field("format", "yyyy-MM-dd HH:mm:ss").field("store", "no").endObject()

```
 
很简单，只需要上面2步，就完成了分词器的设置。
