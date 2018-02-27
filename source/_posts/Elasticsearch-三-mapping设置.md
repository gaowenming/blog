---
title: Elasticsearch(三) mapping设置
date: 2018-02-06 17:39:03
tags: Elasticsearch
categories: Elasticsearch
---


> 之前介绍了Elasticsearch中的数据类型，那如何来定义数据类型呢？这就需要用到Mapping了。
<!--more-->

Mapping是什么
---
> 通俗的解释，mapping和关系型数据库中的schema类似，就是对字段数据存储的描述，包括字段的类型，是否需要分词，怎么分词等。
先看一个index中type的mapping说明
```json
{
    "smart":{
        "mappings":{
            "blog":{
                "properties":{
                    "author":{
                        "type":"keyword",
                        "store":true
                    },
                    "categroy":{
                        "type":"keyword",
                        "store":true
                    },
                    "content":{
                        "type":"text",
                        "store":true,
                        "analyzer":"ik_max_word"
                    },
                    "createTime":{
                        "type":"long",
                        "store":true
                    },
                    "id":{
                        "type":"long",
                        "store":true
                    },
                    "lastUpdateTime":{
                        "type":"date",
                        "store":true,
                        "format":"yyyy-MM-dd HH:mm:ss"
                    },
                    "tags":{
                        "type":"keyword",
                        "store":true
                    },
                    "title":{
                        "type":"text",
                        "store":true,
                        "analyzer":"ik_max_word"
                    }
                }
            }
        }
    }
}
```
主要的属性如下：

 **1. type**
 > type是指该属性的类型，之前也介绍过了，可以根据属性的值设置合适的类型

 **2. index**
 > index是设置该属性是否需要分词，默认的分词是把中文拆成每个汉字，不友好，我们可以设置自己的分词方式，比如ik_smart,ik_max_word

 **3. boost**
 > boost表示该属性的权重，值越大，权重越高

 **4. store**
 > store 的意思是，是否在 _source 之外在独立存储一份，这里要说一下 _source 这是源文档，当你索引数据的时候， elasticsearch 会保存一份源文档到 _source ，如果文档的某一字段设置了 store 为 yes (默认为 no)，这时候会在 _source 存储之外再为这个字段独立进行存储，这么做的目的主要是针对内容比较多的字段，放到 _source 返回的话，因为_source 是把所有字段保存为一份文档，命中后读取只需要一次 IO，包含内容特别多的字段会很占带宽影响性能，通常我们也不需要完整的内容返回(可能只关心摘要)，这时候就没必要放到 _source 里一起返回了(当然也可以在查询时指定返回字段)。
> 
**对内容太长的字段，将 store 设置为 yes ，一般来说还应该在 _source 排除 exclude 掉这个字段，这时候索引的字段，不会保存在 _source 里了，会独立存储一份，查询时 _source 里也没有这个字段了，但是还是可以通过指定返回字段来获取，但是会有额外的 IO 开销，因为 _source 的读取只有一次 IO ，而已经 exclude 并设置 store 的字段，是独立存储的需要一个新的 IO 。**




