### 1、Get API

get Api 通过 document id 来从 index 获取 document。

```shell
GET twitter/_doc/0
```

结果：

```json
{
    "_index" : "twitter",
    "_type" : "_doc",
    "_id" : "0",
    "_version" : 1,
    "_seq_no" : 10,
    "_primary_term" : 1,
    "found": true,
    "_source" : {
        "user" : "kimchy",
        "date" : "2009-11-15T14:12:12",
        "likes": 0,
        "message" : "trying out Elasticsearch"
    }
}
```

结果包含了想要获取的document 的 `_index`、`_type`、`_id`、`_version`，如果能够获取到（`found` 为 `true`） document 则会有document 的实际 `_source`。

使用 `HEAD` 请求可以检验 document 是否存在：

```shell
HEAD twitter/_doc/0
```

#### 1.1、实时的

默认，get API 是实时的，不会受 index 刷新频率的影响。如果 document 被更新，但是还没有刷新，get API 会发起一个刷新来使 document 可见。这会导致上次刷新之后改变的document 变得可见。如果要关闭实时 GET，设置 `realtime` 参数为 `false`。

#### 1.2、Routing

如果 indexing 时使用了`routing`，为了取到 document，也应该指定 `routing`：

```shell
GET twitter/_doc/2?routing=user1
```

如果指定了错误的 `routing` 将取不到 document。

#### 1.3、版本支持

可以使用 `version` 参数获取指定版本的 document。

在内部，ES 标记旧的 document 为deleted，并添加一个全新的document。旧版本的 document 不会马上消失，尽管不能访问它。index 更多数据时，ES 会在后台清理deleted documents。

### 2、Delete API

根据指定的 document id 删除 document。

```console
DELETE /twitter/_doc/1
```

结果：

```json
{
    "_shards" : {
        "total" : 2,
        "failed" : 0,
        "successful" : 2
    },
    "_index" : "twitter",
    "_type" : "_doc",
    "_id" : "1",
    "_version" : 2,
    "_primary_term": 1,
    "_seq_no": 5,
    "result": "deleted"
}
```

#### 2.1、版本

每个被indexed的 document 都是有版本的。删除 document 时，支持提供 `version` 参数，这样可以确保被删除的 document 是想要删除的，并确保与此同时 document 没有被改变。对 document 的写操作，包括删除操作都会导致 document 版本增加。

#### 2.2、Routing

indexing document 时如果使用了 `routing`，删除 document 时也应该提供 `routing`：

```shell
DELETE /twitter/_doc/1?routing=kimchy
```

`routing` 错误，删除会失败。

#### 2.3、刷新

支持刷新 `?refresh` 参数。

### 3、Delete By Query API

按照查询删除 `_delete_by_query`，删除匹配查询的所有 document：

```shell
POST twitter/_delete_by_query
{
  "query": { 
    "match": {
      "message": "some message"
    }
  }
}
```

查询内容必须作为 `query` 的值传递，查询与 Search API 相同。

```json
{
  "took" : 147,
  "timed_out": false,
  "deleted": 119,
  "batches": 1,
  "version_conflicts": 0,
  "noops": 0,
  "retries": {
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,
  "requests_per_second": -1.0,
  "throttled_until_millis": 0,
  "total": 119,
  "failures" : [ ]
}
```

`_delete_by_query` 启动时会获取 index 的快照，使用 `internal` 版本执行删除。如果期间 document 变化，则会有版本冲突。匹配的版本才会被删除。

`_delete_by_query` 是边查询边删除，一个批次一个批次的执行的。删除失败会重试，重试次数超过限制则放弃该批次并将失败计入响应结果的 `failures`。

可以一次性删除多个 index 的多个 type 的 documents：

```shell
POST twitter,blog/_docs,post/_delete_by_query
{
  "query": {
    "match_all": {}
  }
}
```

也支持 `routing` 参数。

#### 3.1、搭配Task API 使用

使用 Task API 可以获取任意执行中的 `_delete_by_query` 的状态：

```shell
GET _tasks?detailed=true&actions=*/delete/byquery
```

响应结果：

```json
{
  "nodes" : {
    "r1A2WoRbTwKZ516z6NEs5A" : {
      "name" : "r1A2WoR",
      "transport_address" : "127.0.0.1:9300",
      "host" : "127.0.0.1",
      "ip" : "127.0.0.1:9300",
      "attributes" : {
        "testattr" : "test",
        "portsfile" : "true"
      },
      "tasks" : {
        "r1A2WoRbTwKZ516z6NEs5A:36619" : {
          "node" : "r1A2WoRbTwKZ516z6NEs5A",
          "id" : 36619,
          "type" : "transport",
          "action" : "indices:data/write/delete/byquery",
          "status" : {    
            "total" : 6154,
            "updated" : 0,
            "created" : 0,
            "deleted" : 3500,
            "batches" : 36,
            "version_conflicts" : 0,
            "noops" : 0,
            "retries": 0,
            "throttled_millis": 0
          },
          "description" : ""
        }
      }
    }
  }
}
```

用 task id 来查看task：

```shell
GET /_tasks/r1A2WoRbTwKZ516z6NEs5A:36619
```

还可以取消 task：

```shell
POST _tasks/r1A2WoRbTwKZ516z6NEs5A:36619/_cancel
```

