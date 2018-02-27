---
title: Elasticsearch(六) 使用Jest实现基本的索引操作和搜索
date: 2018-02-06 17:43:02
tags: Elasticsearch
categories: Elasticsearch
---



> 本文主要是Jest Api的实例介绍，包括index的创建，mapping的设置，基本的条件搜索等。本文中的实现是基于SpringBoot，而且SpringBoot中已经内置了Jest的starter，所以配置起来还是很简单的。
<!--more-->

配置Elasticsearch
---
SpringBoot中已经内置了Elasticsearch的starter，我们只需要在Properties文件中配置Elasticsearch的连接信息即可
```properties
#jest
spring.elasticsearch.jest.uris=http://127.0.0.1:9200
spring.elasticsearch.jest.read-timeout=10000
spring.elasticsearch.jest.username=elastic
spring.elasticsearch.jest.password=changme
```

index通用接口
---
> 1、首先抽象出一个公共类，该类中的方法是对index的基本操作(索引创建、删除、优化、mapping创建、mapping追加)，与index中的数据无关，CommonJestIndexService

```java
package com.smart.elasticsearch.core.base;

import com.google.gson.JsonObject;

import org.elasticsearch.common.xcontent.XContentBuilder;
import org.elasticsearch.common.xcontent.XContentFactory;
import org.springframework.stereotype.Service;

import java.io.IOException;

import javax.annotation.Resource;

import io.searchbox.client.JestClient;
import io.searchbox.client.JestResult;
import io.searchbox.client.JestResultHandler;
import io.searchbox.indices.ClearCache;
import io.searchbox.indices.CreateIndex;
import io.searchbox.indices.DeleteIndex;
import io.searchbox.indices.IndicesExists;
import io.searchbox.indices.Optimize;
import io.searchbox.indices.mapping.GetMapping;
import io.searchbox.indices.mapping.PutMapping;
import lombok.extern.slf4j.Slf4j;

/**
 * 索引操作
 *
 * @author gaowenming
 * @create 2018-02-01 17:45
 * @desc
 **/
@Service
@Slf4j
public class CommonJestIndexService {
    @Resource
    private JestClient jestClient;

    /**
     * 创建index
     */
    public void createIndex(String index) {
        try {
            JestResult jestResult = jestClient.execute(new CreateIndex.Builder(index).build());
            log.info("createIndex:{}", jestResult.isSucceeded());
        } catch (IOException e) {
            log.error("createIndex error:", e);
        }
    }

    /**
     * （设置数据类型和分词方式）
     *
     * 设置index的mapping
     */
    public void createIndexMapping(String index, String type, String mappingString) {
        PutMapping.Builder builder = new PutMapping.Builder(index, type, mappingString);
        try {
            JestResult jestResult = jestClient.execute(builder.build());
            log.info("createIndexMapping result:{}", jestResult.isSucceeded());
            if (!jestResult.isSucceeded()) {
                log.error("settingIndexMapping error:{}", jestResult.getErrorMessage());
            }
        } catch (IOException e) {
            log.error("settingIndexMapping error:", e);
        }
    }

    /**
     * 追加mapping字段
     *
     * @param index     index名称
     * @param type      type名称
     * @param fieldName 字段名称
     * @param fieldType 字段类型
     * @param analyze   是否分词
     */
    public void addFieldMapping(String index, String type, String fieldName, String fieldType, boolean analyze) {
        String mapping;
        XContentBuilder mapBuilder = null;
        try {
            mapBuilder = XContentFactory.jsonBuilder();
            //设置分词
            if (analyze) {
                mapBuilder.startObject()
                        .startObject(type)
                        .startObject("properties")
                        .startObject(fieldName).field("type", fieldType).field("analyzer", "ik_max_word").field("store", "yes").endObject()
                        .endObject()
                        .endObject()
                        .endObject();
            } else {
                mapBuilder.startObject()
                        .startObject(type)
                        .startObject("properties")
                        .startObject(fieldName).field("type", fieldType).field("index", "not_analyzed").field("store", "yes").endObject()
                        .endObject()
                        .endObject()
                        .endObject();
            }
            mapping = mapBuilder.string();
            PutMapping.Builder builder = new PutMapping.Builder(index, type, mapping);
            JestResult jestResult = jestClient.execute(builder.build());
            log.info("addFieldMapping result:{}", jestResult.isSucceeded());
            if (!jestResult.isSucceeded()) {
                log.error("addFieldMapping error:{}", jestResult.getErrorMessage());
            }
        } catch (IOException e) {
            log.error("addFieldMapping error", e);
        }

    }


    /**
     * 追加mapping字段
     *
     * @param index     index名称
     * @param type      type名称
     * @param fieldName 字段名称
     * @param format    日期格式
     */
    public void addDateFieldMapping(String index, String type, String fieldName, String format) {
        String mapping;
        XContentBuilder mapBuilder = null;
        try {
            mapBuilder = XContentFactory.jsonBuilder();

            mapBuilder.startObject()
                    .startObject(type)
                    .startObject("properties")
                    .startObject(fieldName).field("type", "date").field("format", format).field("store", "yes").endObject()
                    .endObject()
                    .endObject()
                    .endObject();

            mapping = mapBuilder.string();
            PutMapping.Builder builder = new PutMapping.Builder(index, type, mapping);
            JestResult jestResult = jestClient.execute(builder.build());
            log.info("addDateFieldMapping result:{}", jestResult.isSucceeded());
            if (!jestResult.isSucceeded()) {
                log.error("addDateFieldMapping error:{}", jestResult.getErrorMessage());
            }
        } catch (IOException e) {
            log.error("addDateFieldMapping error", e);
        }

    }

    /**
     * 获取index的mapping
     */
    public String getMapping(String indexName, String typeName) {
        GetMapping.Builder builder = new GetMapping.Builder();
        builder.addIndex(indexName).addType(typeName);
        try {
            JestResult result = jestClient.execute(builder.build());
            if (result != null && result.isSucceeded()) {
                return result.getSourceAsObject(JsonObject.class).toString();
            }
        } catch (Exception e) {
            log.error("getMapping error", e);
        }
        return null;
    }


    /**
     * 判断index是否存在
     */
    public boolean indexExist(String index) {
        IndicesExists indicesExists = new IndicesExists.Builder(index).build();
        try {
            JestResult jestResult = jestClient.execute(indicesExists);
            if (jestResult != null) {
                return jestResult.isSucceeded();
            }
        } catch (IOException e) {
            log.error("indexExist error", e);
        }
        return false;
    }

    /**
     * 删除index
     */
    public void deleteIndex(String index) {
        try {
            JestResult jestResult = jestClient.execute(new DeleteIndex.Builder(index).build());
            log.info("deleteIndex result:{}", jestResult.isSucceeded());
        } catch (IOException e) {
            log.error("deleteIndex error", e);
        }

    }

    /**
     * 索引优化
     */
    public void optimizeIndex() {
        Optimize optimize = new Optimize.Builder().build();
        jestClient.executeAsync(optimize, new JestResultHandler<JestResult>() {
            @Override
            public void completed(JestResult jestResult) {
                log.info("optimizeIndex result:{}", jestResult.isSucceeded());
            }

            @Override
            public void failed(Exception e) {
                log.error("optimizeIndex error", e);
            }
        });
    }

    /**
     * 清理缓存
     */
    public void clearCache() {
        try {
            ClearCache clearCache = new ClearCache.Builder().build();
            jestClient.execute(clearCache);
        } catch (IOException e) {
            log.error("clearCache error", e);
        }
    }

}

```

数据操作接口
---
> 定义泛型接口，进行index中数据的操作，比如添加索引数据，删除索引数据，简单查询

```java
package com.smart.elasticsearch.core.base;

import com.smart.elasticsearch.core.result.SmartSearchResult;

import java.util.List;

/**
 * 索引中数据的操作（增删改查）
 *
 * @author gaowenming
 * @create 2018-01-31 11:43
 * @desc jestSearch
 **/
public interface JestDataBaseService<T> {

    /**
     * 单条删除
     */
    boolean deleteItem(String index, String type, String id);


    /**
     * 批量创建索引
     */
    void batchIndex(String index, String type, List<T> list);

    /**
     * 单条索引(新增/更新)
     */
    void singleIndex(String index, String type, T t);

    /**
     * 指定索引ID
     */
    void singleIndexWithId(String index, String type, String id, T t);

    /**
     * 根据id查询
     */
    T queryById(String index, String type, String id, Class<T> clazz);

    /**
     * 无过滤条件
     */
    SmartSearchResult<T> queryAll(String index, String type, int fetchSize, Class<T> clazz);

    /**
     * 设置type的mapping
     */
    String buildIndexMapping(String type);

}

```

抽象实现
---
> 泛型接口定义完，我们需要实现其中的方法了，这里我们采用抽象类的实现方式，这样实现的目的是为了我们在定义其他接口时，直接extend该抽象类，即可复用其中的实现。

```java
package com.smart.elasticsearch.core.base;

import com.smart.elasticsearch.core.result.SmartSearchResult;
import com.smart.elasticsearch.core.result.SmartSearchResultConverter;

import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.search.sort.SortOrder;

import java.io.IOException;
import java.util.List;

import javax.annotation.Resource;

import io.searchbox.client.JestClient;
import io.searchbox.client.JestResult;
import io.searchbox.core.Bulk;
import io.searchbox.core.Delete;
import io.searchbox.core.Get;
import io.searchbox.core.Index;
import io.searchbox.core.Search;
import io.searchbox.core.SearchResult;
import lombok.extern.slf4j.Slf4j;

/**
 * elasticsearch通用操作
 *
 * @author gaowenming
 * @create 2018-02-01 14:21
 * @desc
 **/
@Slf4j
public abstract class AbstractJestDataBaseService<T> implements JestDataBaseService<T> {

    @Resource
    private JestClient jestClient;

    @Override
    public boolean deleteItem(String index, String type, String id) {
        try {
            JestResult jestResult = jestClient.execute(new Delete.Builder(id).index(index).type(type).refresh(true).build());
            if (!jestResult.isSucceeded()) {
                log.error("deleteItem error:{}", jestResult.getErrorMessage());
            }
            return jestResult.isSucceeded();
        } catch (IOException e) {
            log.error("deleteItem error", e);
        }
        return false;
    }


    @Override
    public void batchIndex(String index, String type, List<T> list) {
        try {
            Bulk.Builder builder = new Bulk.Builder();
            for (T t : list) {
                builder.addAction(new Index.Builder(t).index(index).type(type).build());
            }
            JestResult jestResult = jestClient.execute(builder.build());
            if (!jestResult.isSucceeded()) {
                log.error("batchIndex error:{}", jestResult.getErrorMessage());
            }
        } catch (IOException e) {
            log.error("batchIndex error", e);
        }
    }

    @Override
    public void singleIndex(String index, String type, T t) {
        try {
            JestResult jestResult = jestClient.execute(new Index.Builder(t).index(index).type(type).build());
            if (!jestResult.isSucceeded()) {
                log.error("singleIndex error:{}", jestResult.getErrorMessage());
            }
        } catch (IOException e) {
            log.error("singleIndex error", e);
        }
    }

    @Override
    public T queryById(String index, String type, String id, Class<T> clazz) {
        T result = null;
        try {
            Get get = new Get.Builder(index, id).type(type).build();
            JestResult jestResult = jestClient.execute(get);
            result = jestResult.getSourceAsObject(clazz);
        } catch (IOException e) {
            log.error("queryById error", e);
        }
        return result;
    }

    @Override
    public void singleIndexWithId(String index, String type, String id, T t) {
        try {
            JestResult jestResult = jestClient.execute(new Index.Builder(t).index(index).type(type).id(id).build());
            if (!jestResult.isSucceeded()) {
                log.error("singleIndexWithId error:{}", jestResult.getErrorMessage());
            }
        } catch (IOException e) {
            log.error("singleIndexWithId error", e);
        }
    }

    @Override
    public SmartSearchResult<T> queryAll(String index, String type, int fetchSize, Class<T> clazz) {
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        searchSourceBuilder.query(QueryBuilders.matchAllQuery()).size(fetchSize).sort("id", SortOrder.DESC);
        Search search = new Search.Builder(searchSourceBuilder.toString())
                .addIndex(index).addType(type)
                .build();
        log.info("search query:{}", searchSourceBuilder);
        SearchResult result;
        SmartSearchResult<T> smartSearchResult = null;
        try {
            result = jestClient.execute(search);
            smartSearchResult = SmartSearchResultConverter.searchResultFormatter(result, clazz);
        } catch (IOException e) {
            log.error("queryAll error", e);
        }
        return smartSearchResult;
    }

}

```

Blog实例
---

>  上面几个类，都是从底层实现的角度来封装Elasticsearch的操作，下面我们用一个实际的例子来应用，加入我们把博客内容存放在Elasticsearch中。

```java
package com.smart.elasticsearch.blog.service;

import com.smart.elasticsearch.blog.domain.Blog;
import com.smart.elasticsearch.blog.param.SearchParam;
import com.smart.elasticsearch.core.base.JestDataBaseService;
import com.smart.elasticsearch.core.result.SmartSearchResult;

/**
 * @author gaowenming
 * @create 2018-01-31 11:43
 * @desc jestSearch
 **/
public interface BlogJestService extends JestDataBaseService<Blog> {

    /**
     * 通用查询
     */
    SmartSearchResult<Blog> queryBySearchParam(SearchParam searchParam);
}

```
> 定义了BlogJestService，继承了泛型接口，再新加了一个通用的搜索接口，基本上对Blog的索引操作都包含了

```java
package com.smart.elasticsearch.blog.service;

import com.smart.elasticsearch.blog.domain.Blog;
import com.smart.elasticsearch.blog.param.SearchParam;
import com.smart.elasticsearch.blog.query.BlogSearchRequest;
import com.smart.elasticsearch.core.base.AbstractJestDataBaseService;
import com.smart.elasticsearch.core.result.SmartSearchResult;

import org.elasticsearch.common.xcontent.XContentBuilder;
import org.elasticsearch.common.xcontent.XContentFactory;
import org.springframework.stereotype.Service;

import java.io.IOException;

import javax.annotation.Resource;

import lombok.extern.slf4j.Slf4j;

/**
 * @author gaowenming
 * @create 2018-02-01 15:13
 * @desc
 **/
@Slf4j
@Service("blogJestService")
public class BlogJestServiceImpl extends AbstractJestDataBaseService<Blog> implements BlogJestService {

    @Resource
    private BlogSearchRequest blogSearchRequest;

    @Override
    public String buildIndexMapping(String type) {
        String mapping = "";
        XContentBuilder mapBuilder = null;
        try {
            mapBuilder = XContentFactory.jsonBuilder();
            mapBuilder.startObject()
                    .startObject(type)
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
                    .endObject()
                    .endObject()
                    .endObject();
            mapping = mapBuilder.string();
        } catch (IOException e) {
            log.error("buildIndexMapping error", e);
        }
        return mapping;
    }

    @Override
    public SmartSearchResult<Blog> queryBySearchParam(SearchParam searchParam) {
        return blogSearchRequest.buildSearchRequest(searchParam);
    }
}

```
> Blog的实现类也很简单，继承AbstractJestDataBaseService抽象类，只需要实现未实现的接口即可。上面的实现中，我们对设置Mapping的方法进行了实现，对每个字段的mapping都进行设置。
 
 
总结
---
> Elasticsearch的Jest Api进行简单的封装，把基本的，通用的操作封装到一起实现，比如我们Elasticsearch中存放Blog和User，都有index的创建，删除等基本操作，抽象出来，对代码的整洁性，可维护性也是很有好处的，我们在Elasticsearch中增加一个新的index，只需要关注mapping和search方面的接口，其他的操作都抽象成通用的方法，也很大的节省了重复的工作。

 


