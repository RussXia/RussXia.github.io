---
layout: blog
title: "Elasticsearch中常用的API"
catalog: true
tag: [Elasticsearch,2019]
---
# 查看集群健康
查看集群信息，检查集群健康,
~~~
GET /_cat/health
~~~
响应信息:
~~~
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1554278556 08:02:36  elasticsearch yellow          1         1     10  10    0    0       10             0                  -                 50.0%
~~~

Green 表示一切正常(集群功能齐全),yellow 表示所有数据可用，但是有些副本尚未分配（集群功能齐全），red 意味着由于某些原因有些数据不可用。注意，集群是 red，它仍然具有部分功能（例如，它将继续从可用的分片中服务搜索请求），但是您可能需要尽快去修复它，因为您已经丢失数据了。


获取我们集群的节点列表
~~~
GET /_cat/nodes
~~~


# 索引的增删查

列出所有的索引:
~~~
GET /_cat/indices
~~~

![查询所有的索引](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/es_api_index_list.jpg)


创建索引:
```json
//创建一个名叫hello的索引,指定分片数量为3，副本数量为2
PUT hello
{
    "settings" : {
        "index" : {
            "number_of_shards" : 3, 
            "number_of_replicas" : 2 
        }
    }
}

//response
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "hello"
}
```

删除索引:
~~~
## 删除hello索引
DELETE /hello

## response
{"acknowledged":true}
~~~

# 文档相关操作

创建文档，索引一条数据到customer索引中，id为1。如果索引操作尚未创建，则索引操作自动创建索引。
7.0.0将会调整默认的分片数量5->1,如果要设置对应的分片数量和副本数量，最好还是先创建对象索引，指定相应的参数。
~~~
PUT /customer/_doc/1
{
  "name": "John Doe"
}
~~~

response:
~~~
{
    "_index": "customer",
    "_type": "_doc",
    "_id": "1",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 1,
    "_primary_term": 1
}
~~~

类似的，修改提交方法，即可查询、删除、修改对应记录。(put和post都可以创建或修改记录)

带上_update API，当对应ID的doc不存在时，会报错而不是直接创建。
~~~
POST /customer/_doc/1/_update
{
  "doc": { "name": "Jane Doe" }
}
~~~

~~~
{
  "error" : {
    "root_cause" : [
      {
        "type" : "document_missing_exception",
        "reason" : "[_doc][9]: document missing",
        "index_uuid" : "NQwB-KspR8qxo2U14Z_-vA",
        "shard" : "1",
        "index" : "customer"
      }
    ],
    "type" : "document_missing_exception",
    "reason" : "[_doc][9]: document missing",
    "index_uuid" : "NQwB-KspR8qxo2U14Z_-vA",
    "shard" : "1",
    "index" : "customer"
  },
  "status" : 404
}
~~~
如果不指定ID，将会自动生成ID。默认情况下，es是根据ID的hash来路由document的，可以设置?routing=来设置hash值。
```json
POST hello/_doc
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}

//response
{
  "_index" : "hello",
  "_type" : "_doc",
  "_id" : "3T8A_GkBPMJabw0lbsBa", //自动生成的ID
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 1,
  "_primary_term" : 1
}

```

每当我们对document做一次更新，Elasticsearch 删除旧的文档，然后一次性应用更新，索引一个新文档。

# 批处理
es提供了[_bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/docs-bulk.html)来实现批处理功能。

批量更新/创建
~~~
POST /customer/_doc/_bulk?pretty
{"index":{"_id":"1"}}
{"name": "John Doe" }
{"index":{"_id":"2"}}
{"name": "Jane Doe" }
~~~
更新ID为1的记录，删除ID为2的记录
~~~
POST /customer/_doc/_bulk?pretty
{"update":{"_id":"1"}}
{"doc": { "name": "John Doe becomes Jane Doe" } }
{"delete":{"_id":"2"}}
~~~
_bulk API支持四种类型的action,`index`,`create`,`delete`和`update`。 index 和 create都以下一行数据作为源，且语义相近。

# 搜索
_search API提供了搜索的相关操作，
~~~
GET /bank/_search?q=*&sort=account_number:asc
~~~

![es_search_result](https://raw.githubusercontent.com/RussXia/RussXia.github.io/master/_pic/es_api_search_doc.jpg)
+ took - Elasticsearch 执行搜索的时间（毫秒）
+ time_out - 告诉我们搜索是否超时
+ _shards - 告诉我们多少个分片被搜索了，以及统计了成功/失败的搜索分片
+ hits - 搜索结果
+ hits.total - 搜索结果
+ hits.hits - 实际的搜索结果数组（默认为前 10 的文档）
+ sort - 结果的排序 key（键）（没有则按 score 排序）
+ score - 搜索后匹配的文档相关度的评估值，分数越高，相关度越高


和上面相同的搜索，使用请求体的方式
~~~
GET /bank/_search
{
  "query": {"match_all": {}},
  "sort": [
    {
      "account_number": {
        "order": "asc"
      }
    }
  ],
  "_source": ["account_number", "address","age","email"],
  "from": 10, 
  "size": 20
}
~~~

+ _source:指定需要返回的字段
+ from:指定开始的行，可以和size配合做分页查询
+ size:查询条数

match_all:匹配所有文档
match:查询包含"mill"或者包含"lane"的字符串(有其中一个就可以)
match_phrase:查询"mill lane"字符串
~~~
  "query": {"match_all": {}} 
  "query": { "match": { "address": "mill lane" } }
  "query": { "match_phrase": { "address": "mill lane" } }
~~~

<B>bool query</B>:
~~~
## 查询age为40，state不为ID的记录
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
~~~
+ must:A && B ... == true,return
+ should: A &#124;&#124; B ... == true,return
+ must_not: !A && !B ... == true,return


bool query也支持filter
~~~
## 查询年纪为21，收入在(20000~30000)之间的记录。
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {
          "age": "21"
        }}
      ], 
      "filter": {
        "range": {
          "balance": {
            "gt": 20000,
            "lt": 30000
          }
        }
      }
    }
  }
}
~~~

# 聚合
以state对account做分组
~~~
GET /bank/_search
{
  "size": 1,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
~~~
ES提供聚合的能力，与sql中的group类似，上述语句类似于
```SQL
SELECT state, COUNT(*) FROM bank GROUP BY state ORDER BY COUNT(*) DESC
```
输入结果,hits是返回的查询结果(size:1),aggregations聚合结果(默认按照count结果降序)
~~~
{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 1000,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "_doc",
        "_id" : "25",
        "_score" : 1.0,
        "_source" : {
          "account_number" : 25,
          "balance" : 40540,
          "firstname" : "Virginia",
          "lastname" : "Ayala",
          "age" : 39,
          "gender" : "F",
          "address" : "171 Putnam Avenue",
          "employer" : "Filodyne",
          "email" : "virginiaayala@filodyne.com",
          "city" : "Nicholson",
          "state" : "PA"
        }
      }
    ]
  },
  "aggregations" : {
    "group_by_state" : {
      "doc_count_error_upper_bound" : 20,
      "sum_other_doc_count" : 770,
      "buckets" : [
        {
          "key" : "ID",
          "doc_count" : 27
        },
        {
          "key" : "TX",
          "doc_count" : 27
        },
        ...省略...
        ...省略...
      ]
    }
  }
}
~~~
先按年龄段（20-29岁，30-39岁，和 40-49岁），然后按照性别分组，组内求平均余额。
~~~
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender.keyword"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}
~~~