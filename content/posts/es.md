---
title: "Elasticsearch使用笔记"
date: 2019-10-20T17:23:54+08:00
draft: true
---

Elasticsearch 提供了很多功能，并且有非常详细的文档，基本上所有的操作都能在官方的文档中找到答案，可以参考 https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html 的介绍。

 

# 下面记录一些使用过程中的notes

当用户把文档存入es中，如何正确的将特定的文档检索出来呢？

Elasticsearch提供了一组restful的API为用户所用，可以通过terminal或者kibana上面提供的Dev Tools工具来进行检索，二者本质上是一样的，后者使用起来更为方便。

使用GET index/_search的操作可以查询文档，下面是一个检索的例子。
```
GET index/_search
{
  "_source": false,
  "timeout": "5s",
  "query": {
    "bool": {
        "must":[
            {
                "match":{
                    "field":"value"
                },
                "wildcard":{
                    "field":"*value*"
                },
                "exists":{
                    "field":"value"
                }
            }
        ],
        "filter":{
            "range":{
                "field":{
                    "gte":157xxx,
                    "lte":158xxx
                }
            }
        }
    }
  },
  "highlight":{
    "fields":{
        "fields":{
            "field1":{},
            "field2":{}
        }
    }
  },
  "size":1000,
  "from":0,
  "sort":{
    "field":{
        "order":"asc",
        "mode":"avg"
    }
  }
}
```

其中，“_source"字段指定在搜索结果中是否显示document的"_source"字段，也可以定制化需要指定显示哪些字段。

 

如何删除调试过程产生的脏数据？

在开发过程中，会产生一些脏数据，但是由于某些原因，这些脏数据会被检索出来影响正常的调试使用。这种情况下可以通过POST index/_delete_by_query来删除指定的数据，下面是一个例子。注意要检查query能够准确匹配到要删除的数据，避免误删除。

```
POST index/_delete_by_query
{
    "query":{
        "match":{
            "field":"value"
        }
    }
}
```

如何更新一条记录？

有时候想要更新一条已经写入es的document，Elasticsearch提供了UPDATE的API来实现这个事情，通过POST index/_update_by_query来实现，下面是一个例子

```
POST kubbiz-audit-service-*/_update_by_query
{
  "query":{
    "match":{
      "_id":"rPth724BMMu1LgcKLt5u"
    }
  },
  "script":{
    "source":"ctx._source.tags.errorCode = 404",
    "lang": "painless"
  }
}
```

再举一个例子

```
POST kubbiz-audit-service-*/_update_by_query
{
  "timeout": "5s",
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "tags.userIdentity.sessionContext.id": "APP_PORTAL_S_9v8YxbGDqNnQPMxEjDuXcknmgN8g6tVq"
          }
        },
        {
          "match": {
            "tags.eventId": "userSignIn15768214904801"
          }
        }
      ]
    }
  },
  "script": {
    "source": "ctx._source.tags.organizationId = params.organizationId;ctx._source.tags.userIdentity.extra.adminLevel = params.adminLevel;",
    "lang": "painless",
    "params": {
      "organizationId": "o15711215352821",
      "adminLevel": "primaryAdmin"
    }
  }
}
```

# 一些有用的tips

在将数据放入es的过程中，value会由analyzer进行tokenized化，然后产生一系列token。在查询的时候会根据这样的token来进行查询。因此，对于一些想要标记特定属性的field来说，最好将其value设置为tokenized过程不变的值，这样有助于筛选，如"helloWorld"要比“hello-world”要好。

 

# 有关分页的问题

分页查询有两种实现方式：

（1）from+size的方式：

from表示从第几条数据开始，size表示一共取多少条数据；

这种方式适用于数据量比较小的情况，Elasticsearch要求from+size要小于10000，否则会报错，这与内部的实现方式有关：查询时，每个shards都会把from+size条数据放入各自的优先队列，然后各个shards把优先队列的数据汇总到单个节点，经过合并排序，最后挑选出从from开始的size条数据；这种方式在深度分页时（from+size很大时），会带来很大的性能开销，cpu、内存、带宽都将成为问题，因此es限制from+size的最大数字为10000；

（2）scroll方式

scroll方式类似于关系型数据库中的cursor，不适合用来做实时搜索，更适合后台批处理任务，如群发；

使用时，首先通过scroll参数将符合条件的结果筛选出来并缓存，可以把它想象成一个snapshot，在初始化之后的插入、删除、修改都不会再影响后续的遍历；使用方式如下：

```
POST index/_search?scroll=1m
{
    "query": { "match_all": {}},
    "size": 10
}
```

然后会返回一个"_scroll_id"，借助这个scroll_id，可以继续遍历后续页码的数据，使用方式如下（注意这一步不需要带index）：

```
POST /_search/scroll
{
"scroll": "1m",
"scroll_id": "your_scroll_id"
}
```

 

可以看出，对于以上两种方式而言，均不适用于数据量非常大（比如10万条以上）的实时分页查询；当然，做这样的需求的时候也需要仔细考虑一下对于如此巨大数量的数据，是否有实时分页的必要。