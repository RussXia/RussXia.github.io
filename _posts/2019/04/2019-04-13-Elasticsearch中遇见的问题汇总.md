---
layout: blog
title: "Elasticsearch中遇到的问题汇总"
catalog: true
tag: [Elasticsearch,踩坑,2019]
---

# 一、聚合操作时，报Fielddata is disabled on text fields by default.
### 报错操作
```
GET /megacorp/employee/_search
{
  "aggs": {
    "all_interests": {
      "terms": {"field": "interests" }
    }
  }
}
```
### 错误reponse:   
```json
{
  "error": {
    "root_cause": [
      {
        "type": "illegal_argument_exception",
        "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [interests] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead."
      }
    ],
    "type": "search_phase_execution_exception",
    "reason": "all shards failed",
    "phase": "query",
    "grouped": true,
    "failed_shards": [
      {
        "shard": 0,
        "index": "megacorp",
        "node": "sNvWT__lQl6p0dMTRaAOAg",
        "reason": {
          "type": "illegal_argument_exception",
          "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [interests] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead."
        }
      }
    ],
    "caused_by": {
      "type": "illegal_argument_exception",
      "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [interests] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead.",
      "caused_by": {
        "type": "illegal_argument_exception",
        "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [interests] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead."
      }
    }
  },
  "status": 400
}
```
### 原因分析
https://www.elastic.co/guide/en/elasticsearch/reference/current/fielddata.html

text类型的字段在查询时使用的是在内存中的称为fielddata的数据结构。这种数据结构是在第一次将字段用于聚合/排序/脚本时基于需求建立的。

它通过读取磁盘上每个segmet上所有的倒排索引来构建，反转term和document的关系(倒排)，并将结果存在Java堆上(内存中)。(因此会耗费很多的堆空间，特别是在加载很高基数的text字段时)。一旦fielddata被加载到堆中，它在segment中的生命周期还是存在的。

因此，加载fielddata是一个非常消耗资源的过程，甚至能导致用户体验到延迟.这就是为什么 fielddata 默认关闭。

开启text 字段的fielddata
```
PUT megacorp/_mapping/employee/
{
  "properties": {
    "interests": { 
      "type":     "text",
      "fielddata": true
    }
  }
}
````

重新聚合
```
GET /megacorp/employee/_search
{
  "aggs": {
    "all_interests": {
      "terms": {"field": "interests" }
    }
  }
}
```
结果正常返回。