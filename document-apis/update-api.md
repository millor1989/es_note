### 1、Update API

 基于提供的脚本或 document 片段，更新指定的 document。从 index 获取 document，执行更新，index 更新后的 document。会使用版本来确保在 “get” 和 “reindex” 期间不会更新。

这个操作意味着 document 的 full reindex

例子：

```shell
PUT test/_doc/1
{
    "counter" : 1,
    "tags" : ["red"]
}
```

#### 1.1、脚本更新

增加 `counter` 值：

```shell
POST test/_doc/1/_update
{
    "script" : {
        "source": "ctx._source.counter += params.count",
        "lang": "painless",
        "params" : {
            "count" : 4
        }
    }
}
```

往 `tags` 集合中增加 tag：

```shell
POST test/_doc/1/_update
{
    "script" : {
        "source": "ctx._source.tags.add(params.tag)",
        "lang": "painless",
        "params" : {
            "tag" : "blue"
        }
    }
}
```

#### 1.2、使用 document 片段更新

```shell
POST test/_doc/1/_update
{
    "doc" : {
        "name" : "new_name"
    }
}
```

如果同时指定了 `doc` 和 `script` 则忽略 `doc`。

### 2、Update By Query API

`_update_by_query` 的最简单应用是对 index 的每个 document 执行更新而不改变 document 的 source。

```shell
POST twitter/_update_by_query?conflicts=proceed
```

与 `_delete_by_query` 类似，`_update_by_query` 也是根据 index 快照进行更新操作。也会有版本冲突的问题。

相比 Update API，增加了 `query` ：

```shell
POST twitter/_update_by_query
{
  "script": {
    "source": "ctx._source.likes++",
    "lang": "painless"
  },
  "query": {
    "term": {
      "user": "kimchy"
    }
  }
}
```

也支持 `routing` 参数。

通知指定一个 `pipeline` `_update_by_query` 还可以使用 Ingest Node 特性：

```shell
PUT _ingest/pipeline/set-foo
{
  "description" : "sets foo",
  "processors" : [ {
      "set" : {
        "field": "foo",
        "value": "bar"
      }
  } ]
}


POST twitter/_update_by_query?pipeline=set-foo
```

#### 2.1、搭配 Task API 使用

使用 Task API 可以获取所有运行中的 update by query 请求的状态：

```shell
GET _tasks?detailed=true&actions=*byquery
```

响应：

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
          "action" : "indices:data/write/update/byquery",
          "status" : {    
            "total" : 6154,
            "updated" : 3500,
            "created" : 0,
            "deleted" : 0,
            "batches" : 4,
            "version_conflicts" : 0,
            "noops" : 0,
            "retries": {
              "bulk": 0,
              "search": 0
            },
            "throttled_millis": 0
          },
          "description" : ""
        }
      }
    }
  }
}
```

查看任务状态：

```shell
GET /_tasks/r1A2WoRbTwKZ516z6NEs5A:36619
```

取消 update by query 任务：

```shell
POST _tasks/r1A2WoRbTwKZ516z6NEs5A:36619/_cancel
```

### 3、Multi Get API

Multi get API 返回多个 documents。响应包含一个 `docs` 的数组——包含了获取到的 documents。

```shell
GET /_mget
{
    "docs" : [
        {
            "_index" : "test",
            "_type" : "_doc",
            "_id" : "1"
        },
        {
            "_index" : "test",
            "_type" : "_doc",
            "_id" : "2"
        }
    ]
}
```

写法可以改为针对某个 index 的：

```shell
GET /test/_mget
{
    "docs" : [
        {
            "_type" : "_doc",
            "_id" : "1"
        },
        {
            "_type" : "_doc",
            "_id" : "2"
        }
    ]
}
```

或者针对 type 的：

```shell
GET /test/_doc/_mget
{
    "docs" : [
        {
            "_id" : "1"
        },
        {
            "_id" : "2"
        }
    ]
}
```

也可以使用如下简化的写法：

```shell
GET /test/_doc/_mget
{
    "ids" : ["1", "2"]
}
```

支持 `routing` 参数。