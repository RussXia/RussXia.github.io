---
layout: blog
title: "Elasticsearch中的并发控制"
catalog: true
tag: [Elasticsearch,2019]
---
# 数据库领域常见的并发控制
+ 悲观并发控制<br>
假定冲突发生概率很高，在读取一行数据前，先锁定这一行，这样就可以确保只有读取到这行数据的线程可以修改这一行数据。
+ 乐观并发控制<br>
这也是ES所使用的。假设冲突发生概率不高，也不会去阻止某一数据的访问。然而，如果基础数据在我们读取和写入的间隔中发生了变化，更新就会失败。这时候就由程序来决定如何处理这个冲突。例如，它可以重新读取新数据来进行更新，又或者它可以将这一情况直接反馈给用户。

# ES中的乐观并发控制
ES是分布式的。当文档发生创建、更新、删除等写操作时，新版本的文档必须复制到集群中的其他节点。Elasticsearch 也是异步和并发的，这意味着这些复制请求被并行发送，并且到达目的地时也许 顺序是乱的 。ES使用 `_seq_no`(sequence number) 和 `_primary_term`(primary term) 来确保文档的旧版本不会覆盖新的版本。sequence number在每次操作后递增,因此新的操作的_seq_no一定高于老的操作的_seq_no。

```
## 创建对应文档
PUT /demo/_doc/1/_create
{
  "name": "ddd",
  "age":  10
}

GET /demo/_doc/1
## response
{
  "_index" : "demo",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "ddd",
    "age" : 10
  }
}
```
返回值中，我们可以看到sequence number 对应的 `_seq_no` 和primary term对应的 `_primary_term`字段。在一批update操作后，`_seq_no` 和 `_primary_term` 的相关value变到了以下状态。
```
{
  "_index" : "demo",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 20,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 19,
  "_primary_term" : 2
}
```
此时可以根据 `_seq_no` 和 `_primary_term` 乐观更新。sequence number 和 the primary term 独一无二地标识一次变动。
```
PUT demo/_doc/1?if_seq_no=19&if_primary_term=2
{
  "name" : "dsadas",
  "age" : 20
}
```
response
```json
{
  "_index" : "demo",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 21,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 20,
  "_primary_term" : 2
}
```
如果并发了，此时可能有一个 _seq_no 仍为19的记录请求进行update，将会报错。
```json
{
  "error": {
    "root_cause": [
      {
        "type": "version_conflict_engine_exception",
        "reason": "[_doc][1]: version conflict, required seqNo [19], primary term [2]. current document has seqNo [20] and primary term [2]",
        "index_uuid": "aTLF5rchS7C1jbRTyMfdgQ",
        "shard": "3",
        "index": "demo"
      }
    ],
    "type": "version_conflict_engine_exception",
    "reason": "[_doc][1]: version conflict, required seqNo [19], primary term [2]. current document has seqNo [20] and primary term [2]",
    "index_uuid": "aTLF5rchS7C1jbRTyMfdgQ",
    "shard": "3",
    "index": "demo"
  },
  "status": 409
}
```