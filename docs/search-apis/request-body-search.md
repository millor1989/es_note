### Request Body Search

可以在请求体中使用检索 DSL （包括 [Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/query-dsl.html)）来执行检索请求。比如：

```shell
GET /twitter/_search
{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

响应为：

```shell
{
    "took": 1,
    "timed_out": false,
    "_shards":{
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
    },
    "hits":{
        "total" : 1,
        "max_score": 1.3862944,
        "hits" : [
            {
                "_index" : "twitter",
                "_type" : "_doc",
                "_id" : "0",
                "_score": 1.3862944,
                "_source" : {
                    "user" : "kimchy",
                    "message": "trying out Elasticsearch",
                    "date" : "2009-11-15T14:12:12",
                    "likes" : 0
                }
            }
        ]
    }
}
```

##### 参数

| 参数            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `timeout`       | 检索超时时间                                                 |
| `from`          | 获取结果的起始偏移量，默认 `0`                               |
| `size`          | 返回命中结果的数量。默认`10`。如果不关心命中的documents，只关注匹配的数量或者聚合，设置为 `0` 可以提升查询性能。 |
| `search_type`   | [Search Type](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/search-request-search-type.html) |
| `request_cache` | 对 `size` 为 0 的请求开启（true）或关闭（false）结果缓存。   |

`search_type`、`request_cache` 和 `allow_partial_search_results` 设置必须作为查询字符串参数传递。其它参数应该通过请求体传递。请求体也可以作为名为 `source` 的 REST 参数传递。

#### 1、Query

请求体中的 `query` 元素使用 [Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/query-dsl.html) 来定义查询。

```shell
GET /_search
{
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

#### 2、From / Size

`from` 和 `size` 参数可以用来分页，它们既可以作为请求参数，也可以用在请求体中。

```shell
GET /_search
{
    "from" : 0, "size" : 10,
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

`from` + `size` 不能超过 index 设置 `index.max_result_window`（默认 10000）。可以使用 Scroll 或 Search After API 进行更高效的滚动查询。

#### 3、Sort

进行属性级别的排序。使用属性名 `_score` 来按 score 排序，使用 `_doc` 属性名来按 index 顺序排序（没有实际的应用场景，但它是最高效的排序顺序）。

假设有如下 index mapping：

```shell
PUT /my_index
{
    "mappings": {
        "_doc": {
            "properties": {
                "post_date": { "type": "date" },
                "user": {
                    "type": "keyword"
                },
                "name": {
                    "type": "keyword"
                },
                "age": { "type": "integer" }
            }
        }
    }
}
```

```shell
GET /my_index/_search
{
    "sort" : [
        { "post_date" : {"order" : "asc"}},
        "user",
        { "name" : "desc" },
        { "age" : "desc" },
        "_score"
    ],
    "query" : {
        "term" : { "user" : "kimchy" }
    }
}
```

响应中会包含排序的值。

`asc` 升序，`desc` 降序。根据 `_score` 排序时，默认降序，其它情况下都是默认升序。

ES 支持对多值属性排序， `mode` 选项控制如果对多值属性排序，`mode` 支持的值为 `min`、`max`、`sum`、`avg`、`median`。例如：

```shell
PUT /my_index/_doc/1?refresh
{
   "product": "chocolate",
   "price": [20, 4]
}

POST /_search
{
   "query" : {
      "term" : { "product" : "chocolate" }
   },
   "sort" : [
      {"price" : {"order" : "asc", "mode" : "avg"}}
   ]
}
```

