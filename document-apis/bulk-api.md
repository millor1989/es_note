### 1、Bulk API

bulk API 可以在一个 API 调用中执行多个 index/delete 操作。可以极大地提高indexing 速度。

REST API 以 `/bulk` 结束，参数需要遵循 NDJSON（newline delimited JSON）结构：

```t
action_and_meta_data\n
optional_source\n
action_and_meta_data\n
optional_source\n
....
action_and_meta_data\n
optional_source\n
```

支持的 action 包括 `index`、`create`、`delete`、`update`。`index` 和 `create` 需要下一行有 source，并且语法和标准 index Api 的`op_type` 参数一样。`delete` 不需要source。 `update` 需要提供 doc 片段、upsert 和 script。

如果使用文本文件作为 `curl` 的输入，必须使用 `--date-binary` 标识，而不是普通文本标识 `-d`（不会保留换行符）。

```shell
$ cat requests
{ "index" : { "_index" : "test", "_type" : "_doc", "_id" : "1" } }
{ "field1" : "value1" }

$ curl -s -H "Content-Type: application/x-ndjson" -XPOST localhost:9200/_bulk --data-binary "@requests"; echo
{"took":7, "errors": false, "items":[{"index":{"_index":"test","_type":"_doc","_id":"1","_version":1,"result":"created","forced_refresh":false}}]}
```

如下为一个 bulk 命令的正确顺序：

```shell
curl -X POST "localhost:9200/_bulk?pretty" -H 'Content-Type: application/json' -d'
{ "index" : { "_index" : "test", "_type" : "_doc", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_type" : "_doc", "_id" : "2" } }
{ "create" : { "_index" : "test", "_type" : "_doc", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_type" : "_doc", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }'
```

结果：

```json
{
   "took": 30,
   "errors": false,
   "items": [
      {
         "index": {
            "_index": "test",
            "_type": "_doc",
            "_id": "1",
            "_version": 1,
            "result": "created",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 201,
            "_seq_no" : 0,
            "_primary_term": 1
         }
      },
      {
         "delete": {
            "_index": "test",
            "_type": "_doc",
            "_id": "2",
            "_version": 1,
            "result": "not_found",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 404,
            "_seq_no" : 1,
            "_primary_term" : 2
         }
      },
      {
         "create": {
            "_index": "test",
            "_type": "_doc",
            "_id": "3",
            "_version": 1,
            "result": "created",
            "_shards": {
               "total": 2,
               "successful": 1,
               "failed": 0
            },
            "status": 201,
            "_seq_no" : 2,
            "_primary_term" : 3
         }
      },
      {
         "update": {
            "_index": "test",
            "_type": "_doc",
            "_id": "1",
            "_version": 2,
            "result": "updated",
            "_shards": {
                "total": 2,
                "successful": 1,
                "failed": 0
            },
            "status": 200,
            "_seq_no" : 3,
            "_primary_term" : 4
         }
      }
   ]
}
```

请求 URL 可以是 `/_bulk`, `/{index}/_bulk`,  `{index}/{type}/_bulk`。

bulk 支持 `routing`、版本、支持设置`refresh`。

### 2、Reindex API

`_reindex` 最基本的形式是将documents 从一个 index 复制到另一个 index：

```shell
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  }
}
```

与 `by_query` API 类似，reindex 操作的也是快照。

`dest` 的 `version_type` 默认`internal`，盲目的复制 documents 到新的index，覆盖目标 index 的 `version`：

```shell
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "internal"
  }
}
```

`version_type` 为 `external` 时 ES 保留源 index 中 document 的 version，如果目标 index 中没有 documents 则创建，已经存在旧版本则更新：

```shell
POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter",
    "version_type": "external"
  }
}
```

TO BE CONTINUED ! ! !

### 3、Term Vectors（词向量）

返回一个特定 document 的属性的词（term）的信息和统计。默认是实时的，可以通过设置 `realtime` 为 `false` 改变默认设置。

还可以通过 url 参数指定获取哪些属性的信息：

```shell
GET /twitter/_doc/1/_termvectors?fields=message
```

### 4、Multi termvectors API

