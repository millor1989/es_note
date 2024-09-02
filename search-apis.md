### Search APIs

大多数的 search APIs 都是支持多 index 的，Explain API 是例外。检索可以通过使用简单的查询字符串作为参数，也可以使用请求体。如果一个或者多个 shards 故障，为了确保快速地响应，search API 会用部分结果来响应。

#### 1、Routing

执行检索时，检索会被广播到所有的 index/indices shards（在 replicas 直接循环）。通过指定 `routing` 参数可以限制检索的 shards。比如，indexing 时使用了 `routing` 参数：

```shell
POST /twitter/_doc?routing=kimchy
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

那么，检索时，就可以使用 `routing`，只检索相关的 shard：

```shell
POST /twitter/_search?routing=kimchy
{
    "query": {
        "bool" : {
            "must" : {
                "query_string" : {
                    "query" : "some query string here"
                }
            },
            "filter" : {
                "term" : { "user" : "kimchy" }
            }
        }
    }
}
```

`routing` 参数可以是**逗号分隔**的多个值。

#### 2、自适应的 Replica 筛选

除了以循环的方式将请求发送到 replica，还可以使用自适应 replica 选择。此时，协调节点会根据如下几个标准将请求发送到被认为是”最佳的“ replica：

- 协调节点和包含数据拷贝的节点之间过去（past）请求的响应时间
- 过去检索请求在包含数据的节点上执行的时间
- 包含数据的节点上检索线程池队列的大小

改变动态集群设置 `cluster.routing.use_adaptive_replica_selection` 为 `true`，开启自适应 replica 选择：

```shell
PUT /_cluster/settings
{
    "transient": {
        "cluster.routing.use_adaptive_replica_selection": true
    }
}
```

#### 3、统计组（Stats Groups）

检索可以与统计组进行关联，每个统计组中维护着一个统计聚合。可以在之后，通过 indices stats API 获取。比如，将请求于两个不同统计组关联：

```shell
POST /_search
{
    "query" : {
        "match_all" : {}
    },
    "stats" : ["group1", "group2"]
}
```

4、Global Search Timeout

5、Search Cancellation

使用 task cancellation 机制可以取消检索。

#### 6、检索并发和并行度

默认情况下，基于请求命中的shards 的数量，ES 不会拒绝任何请求。虽然 ES 会优化在协调节点上执行的检索，但是大量的分片会对 CPU 和内存产生重大影响。更好的做法是将数据组织为少数大的 shards。可以更新集群设置 `action.search.shard_count.limit` 以拒绝命中太多 shards 的请求。

请求参数 `max_concurrent_shard_requests` 可以用来控制 search API 最大的并发分片请求的数量。