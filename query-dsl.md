### Query DSL

Es提供了一个完整的Query DSL（REST Query DSL，ES 的 Java API 提供了与 REST Query DSL 类似的 Java Query DSL），基于JSON来定义查询。可以将 Query DSL 当作一个查询的 AST（Abstract Syntax Tree，抽象语法树），由两类语句组成：

- ##### 叶查询语句（Leaf query clauses）

  叶查询语句在特定的属性中查询特定的值，比如`match`，`term`或者`range`查询。这些查询可以单独使用。

- ##### 混合查询语句（Compound query clauses）

  混合查询语句包括着其它叶查询或者混合查询，用于以一个逻辑风格（比如`bool`查询或者`dis_max`查询）组合多个查询，或者修改它们的行为（比如`constant_score`查询）。

查询语句用在查询上下文（query context）或者过滤上下文（filter context）中时，会有不同的行为。

- ##### 查询上下文

  查询上下文中的查询语句回答的问题是——“这个document和这个查询语句的匹配得怎么样？”

  除了决定document是否匹配，查询语句也计算一个表示document相对其他documents匹配程度的`_score`

  当将查询语句传递给`query`参数，查询上下文就会生效，比如`search` API 中的`query`参数。

- ##### 过滤上下文

  在过滤器上下文中，查询语句回答的问题是——“这个document是否匹配这个查询语句？”，回答是Yes或者No——不会计算scores。过滤器上下文大多用于过滤结构化的数据，比如：

  - `timestamp`是否在范围2015至2016中
  - `status`属性是否是`"published"`

  为了提升性能，频繁使用的过滤器会被Elasticsearch自动地缓存。

  当查询语句被传递给`filter`参数时过滤上下文生效，比如`bool`查询中的`filter`或者`must_not`参数，`constant_score`查询中的`filter`参数，或者`filter`聚合。

如下为查询语句被用在`search` API中的查询上下文和过滤上下文的一个例子。这个查询会匹配满足一下所有条件的documents：

- `title`属性包含`search`
- `content`属性包含`elasticsearch`
- `status`属性为`published`
- `publish_state`属性是一个大于等于2015-01-01的日期

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "title":   "Search"        }}, 
        { "match": { "content": "Elasticsearch" }}  
      ],
      "filter": [ 
        { "term":  { "status": "published" }}, 
        { "range": { "publish_date": { "gte": "2015-01-01" }}} 
      ]
    }
  }
}
'
```

1. `query`参数表示是查询上下文；
2. `bool`和两个`match`语句用在查询上下文中，意味着它们是用于给document的匹配度评分的
3. `filter`参数表示是过滤上下文
4. `term`和`range`语句用在过滤上下文中。它们会过滤掉不匹配的documents，但是不会影响匹配的documents的score。

#### Match All查询

最简单的查询，匹配所有的documents，并给它们的`_score`都是`1.0`。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "match_all": {}
    }
}'
```

使用`boost`参数可以改变`_score`：

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "match_all": { "boost" : 1.2 }
    }
}'
```

#### Match None查询

不匹配任何documents

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "match_none": {}
    }
}'
```

