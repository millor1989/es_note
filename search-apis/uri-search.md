### URI Search

通过提供请求参数可以纯粹使用URI执行检索请求。并不是所有的检索选项都可以通过这种模式执行，但是这种方式便于执行测试。比如：

```shell
GET twitter/_search?q=user:kimchy
```

其中参数 `q` 表示查询字符串（对应于 `query_string` 查询，详情参考[Query String Query](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/query-dsl-query-string-query.html)）。