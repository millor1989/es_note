### Index API

对指定 index 增加或者更新 document，并使 document 可查询。

往 `twitter` index 的 `_doc` type 添加 id 为 1 的 document：

```shell
curl -X PUT "localhost:9200/twitter/_doc/1?pretty" -H 'Content-Type: application/json' -d'{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}'
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
    "_version" : 1,
    "_seq_no" : 0,
    "_primary_term" : 1,
    "result" : "created"
}
```

`_shards` 头提供了 index 操作的 replication 处理信息：

`total` ：表示index 操作对多少个 shard 拷贝执行了操作

`successful`：表示 index 操作在多少个shard 拷贝上成功了

`failed`：index 操作在 replica shard 上失败时，包含相关的错误信息。

当 `successful` 至少为 1 时，index 操作是成功的。

#### 1、自动创建 Index

`action.auto_create_index` 设置控制了 index 的自动创建：

```shell
# 只允许指定模式的index的创建
PUT _cluster/settings
{
    "persistent": {
        "action.auto_create_index": "twitter,index10,-index1*,+ind*" 
    }
}

# 不允许自动创建index
PUT _cluster/settings
{
    "persistent": {
        "action.auto_create_index": "false" 
    }
}

# 可以自动创建任意index，默认设置
PUT _cluster/settings
{
    "persistent": {
        "action.auto_create_index": "true" 
    }
}
```

#### 2、操作类型

index 操作有一个 `op_type` 参数，值为 `create` 时，对应 id 的document 不存在时 index 操作成功，否则失败。

```shell
PUT twitter/_doc/1?op_type=create
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

追加 `_create` 等价与 `op_type=create`：

```shell
PUT twitter/_doc/1/_create
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

#### 3、自动生成 ID

可以自动生成 document id，此时 `op_type` 默认为 `create`：

```shell
POST twitter/_doc/
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

结果中包含生成的id：

```json
{
    "_shards" : {
        "total" : 2,
        "failed" : 0,
        "successful" : 2
    },
    "_index" : "twitter",
    "_type" : "_doc",
    "_id" : "W0tpsmIBdwcYyG50zbta",
    "_version" : 1,
    "_seq_no" : 0,
    "_primary_term" : 1,
    "result": "created"
}
```

#### 4、路由（Routing）

默认，shard 放置（或者说，`routing`）是通过使用document 的 id值得hash控制的。为了更加明确的控制，传给 router 使用的 hash 函数的值可以使用 `routing` 参数直接指定：

```shell
POST twitter/_doc?routing=kimchy
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

#### 5、等待活跃的 Shards

为了提升系统写操作的弹性，可以配置 indexing 操作执行处理前等待一定数量的活跃 shard 拷贝，否则写操作将等待并重试。默认的`wait_for_active_shards=1`。默认值可以在 index 设置中通过设置 `index.write.wait_for_active_shards` 动态地覆盖。

#### 6、Refresh

Index、Update、Delete、Bulk APIs 都支持设置 `refresh` 来控制请求导致的改变何时对检索请求可见。

- `?refresh` 或者 `?refresh=true`

  操作发生后立即刷新相关的（不是整个 index 的） primary 和 replica shards，以便 改变的 document 马上对检索请求可见。只有在确保不会导致性能很差的情况下使用这种刷新方式。

- `?refresh=wait_for`

  做出回应之前确保请求导致的改变通过 refresh 变得可见。不会强制马上刷新，而是等待刷新发生。ES 每隔 `index.refresh_interval` （默认 1 s）自动刷新发生变化的 shards。调用 Index Refresh API（`_refresh`）或者为支持的 API 设置 `?refresh=true` 也会导致刷新，这都会引起已经运行的带有`refresh=wait_for` 的请求做出回应。

- `?refresh=false`

  默认设置。请求导致的改变会在请求返回后的某个时间点对检索请求可见。