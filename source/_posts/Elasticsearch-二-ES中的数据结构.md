---
title: Elasticsearch(二) ES中的数据结构
date: 2018-02-06 17:37:56
tags: Elasticsearch
categories: Elasticsearch
---


> 在关系型数据库中，表中的字段都是需要定义类型的，在Elasticsearch中，同样也有类型的概念。
<!--more-->

核心类型
---
![](/images/datatype.png)

 1. 字符类型
 字符类型包括：text，keyword
>  **text类型**：分词，将大段的文字根据分词器切分成独立的词或者词组，以便全文检索。
**适用**：email内容、描述等需要分词全文检索的字段；
**不适用**：排序或聚合（Significant Terms 聚合例外）
**keyword类型**：无需分词、整段完整精确匹配。
**适用**：email地址、住址、状态码、分类tags。

 2. 数值类型
> long
integer
short 
byte
double
float 
half_float半精度浮点型：半精度16位IEEE 754浮点数。
scaled_float：由长度固定的缩放因子支持的浮点数。
以上，根据长度和精度选型即可。
 3. 日期类型 （date）

 > "date": {
          "type":   "date",
          "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"

}
日期类型需要指定日期的格式，而且es对日期的格式是很敏感的，所以需要严格的按照格式赋值
 4. 布尔值（boolean）
   字段的值为true、false
 5. 二进制
> 二进制类型接受二进制值作为Base64编码字符串。 **该字段默认情况下不存储，不可搜索**。
如： "blob": "U29tZSBiaW5hcnkgYmxvYg=="

复杂类型
---

 1. 数组类型
 > 在Elasticsearch中，没有专门的数组类型。任何类型都可以组成数组类型，但是在数组中的数据，他们的类型必须是一致的，

 2. 对象类型
> JSON文档本质上是分层的：存储类似json具有层级的数据，文档可能包含内部对象，而内部对象又可能包含其他内部对象。

 
 3. 嵌套类型
 >嵌套类型和对象类型有点类似，nested嵌套类型是Object数据类型的特定版本，允许对象数组彼此独立地进行索引和查询。

地理坐标
---

 1. GEO Point
>_ geo_point _ 用于经纬度坐标。
 
 2. GEO shape
>_ geo_shape _ 用于类似于多边形的复杂形状，划定的地理围栏。

特定类型
---

 1. IP
 >IPv4 类型（IPv4 datatype）：_ ip _ 用于IPv4 地址； 
**"ip_addr": "192.168.1.1"**

 2. Completion 类型
>_ completion _提供自动补全建议；

 3. Token count
 >_ token_count _ 用于统计做了标记的字段的index数目，该值会一直增加，不会因为过滤条件而减少。

 4. mapper-murmur3
 >通过插件，可以通过 _ murmur3 _ 来计算 index 的 hash 值；

 5. Attachment datatype
> 附加类型（Attachment datatype）：采用 mapper-attachments
插件，可支持_ attachments _ 索引，例如 Microsoft Office 格式，Open Document 格式，ePub, HTML 等。

