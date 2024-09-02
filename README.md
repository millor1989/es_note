### ElasticSearch

一个高度可扩展的（highly scalable）、开源的、全文检索（full-text search）和分析（analytics）引擎，可以快速地和接近实时地（near real time）储存、搜索和分析海量数据。它通常用作支持具有复杂搜索功能和要求的应用程序的基础引擎/技术。

一些应用场景：

- 网上商城商品搜索。保存完整的商品目录和存货，为用户提供搜索和自动完成建议。
- 收集日志或交易数据，分析和挖掘这些数据的趋势（trends）、统计数据（statistics）、摘要（summarizations）或异常（anomalies）。这种场景下，可以使用Logstash（Elasticsearch/Logstash/Kibana 栈的一部分）收集、聚合和分析数据，然后让Logstash将这些数据发送给Elasticsearch，Elasticsearch就可以对这些数据进行搜索、聚合和挖掘了。
- 运行一个价格提醒平台，用户可以指定规则，比如“想要买一个专用电子工具，并且想要在下一个月中，任何商家的这种工具的价格低于$X时，收到通知”。此时，可以爬取供应商价格，将它们推送给Elasticsearch并使用它的反向检索功能（reverse-search，Percolator）来将用户需求和价格变化进行匹配，最终在匹配的时候对用户发出提醒。
- 分析/商业智能（business-intelligence）需求，并且想要对大量数据进行快速的调查、分析、可视化，以及提出问题。这种情况下，可以使用Elasticsearch保存数据，使用Kibana来构建自定义的dashboard以可视化关心的数据。此外，还可以使用Elasticsearch的聚合功能来对数据进行复杂的BI查询。

ElasticSearch是Lucene的封装，用Java开发的，提供了REST API的操作接口，开箱即用。

安装上，依赖Java，Elastic 默认运行在9200端口，直接访问该端口会得到 Elastic 的信息（包括节点、集群、版本等信息）。

```shell
$ curl localhost:9200

{
  "name" : "atntrTf",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "tf9250XhQ6ee4h7YI11anA",
  "version" : {
    "number" : "5.5.1",
    "build_hash" : "19c13d0",
    "build_date" : "2017-07-18T20:44:24.823Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
}
```

默认情况下，Elastic只允许本机访问，如果需要远程访问，修改Elastic安装目录的`config/ealsticsearch.yml`文件，增加设置“network.host:0.0.0.0”，重启Elastic即可。

健康检查：

```shell
curl -X GET "localhost:9200/_cat/health?v&pretty"
```

green表示良好；yellow表示数据可用，但是备份没完成；red表示某些数据不可用。

即使是red，ES也是部分可用的。

#### 1、基本概念

##### 1.1、Node、Cluster

Elastic本质上是一个**分布式数据库**，允许多台服务器协同工作，每台服务器可运行多个ES实例。
一个ES实例为一个**节点**（**node**），一组节点构成一个**集群**（**cluster**）

##### 1.2、Index

ES会**索引所有字段**，经过处理后写入一个**反向索引**（Inverted Index）。查找数据的时候，直接查找该索引。	
索引（Index）是ES数据管理的顶层单位。它是单个数据库的同义词。每个**Index（数据库）**的名字必须是小写。
查看当前节点的所有Index	

```shell
$ curl -X GET 'http://localhost:9200/_cat/indices?v'
```

##### 1.3、Document

Index里面的**单条记录**称为Document（文档）。许多条Document构成了一个Index。Document使用Json格式表示，同一个Index里面的Document，不要求有相同的结构（schema）；但是如果有相同的结构，有利于提高搜索效率。

尽管Document物理上是保存在index中的，document实际上必须被索引/分配到index中的type中。

反向索引包含了所有 Document 中的唯一词（unique word）并且定位了每个词所在的 documents。

默认情况下，ES 为每个属性的所有数据建立索引，并且每个被索引的属性都有一个专用的、优化的数据结构。比如，text 属性保存在反向索引中，数值和地理属性保存在 BKD 树中。

ES 可以是无格式的，即可以在不明确指定如何对documents 中可能出现的不同属性进行处理的情况下，对documents 进行索引。当开启动态映射（dynamic mapping）时，ES 会自动检测和添加属性到index。

定义映射（mappings）的好处是：

- 区分全文字符串属性和精确值字符串属性
- 进行针对语言的（language-specific）文本分析（text analysis）
- 优化属性的部分匹配（partial matching）
- 使用自定义的日期格式
- 使用不能被自动检测的数据类型（比如，`geo_point`和 `geo_shape`）

对于同一个属性，为了不同的目的经常要进行不同的索引。比如，可能会在将一个字符串属性索引为一个文本属性进行全文检索的同时，将其索引为一个keyword属性以排序和聚合数据。或者，可能会选择使用多个语言分词器（language analyzer）来处理包含用户输入内容的字符串属性。

对全文本属性进行索引时使用的分析链（analysis chain）在检索时也会使用。当查询全文本属性时，首先会对检索文本进行相同的分析，然后才到index中进行查询。

##### 1.4、Type

Document可以分组，这种**分组**就叫做Type（mapping type）；它是虚拟的逻辑分组，用来过滤Document。同一个Index中，不同的Type应该有相似的结构（schema），性质完全不同的数据应该存在两个Index，而不是一个Index的两个Type。
查看每个Index所包含的Type：

```shell
$ curl 'localhost:9200/_mapping?pretty=true'
```

es 5.x 版本返回结果的结构如下：

```json
{
  "sa_index": {
    "mappings": {
      "act": {
        "properties": {
          "appVersion": {
            "type": "text"
          },
          "biz_time": {
            "type": "date"
          },
          "id": {
            "type": "long"
          }
        }
      },
      "carina": {
        "properties": {
          "biz_time": {
            "type": "date"
          },
          "user_id": {
            "type": "long"
          },
          "ums_id": {
            "type": "long"
          }
        }
      }
    }
  }
}
```

根元素为 index 名，`mappings` 下的第一级元素就是 index 包含的 `type`。

习惯上，可能会将 index 比作关系型数据库的数据库，将 type 比作关系型数据库的表，但是这对 ES 是不恰当的。关系型数据库的两个表是相互独立的。而 es index 中不同 mapping types 的同名属性在底层都是相同的 Lucene 属性。所以同一个 index 中不同 types 的同名属性必须有相同的的 mapping 定义。这在相同index不同types的同名属性需要不同定义的时候会有困难。另外，在一个index中保存具有很少或者没有共用属性的记录会导致数据稀疏，而这就干扰了 Lucene 对 document 的高效压缩。因此，根据规划，Elastic 6.x的版本只允许每个Index包含一个Type， 7.X 的版本将会**彻底移除** Type 。7.x 版本的 API 都支持无 type 的请求（6.x 版本已经有某些 API 支持无 type 的请求了），如果指定了 type 反而会有废止警告。

##### 1.5、Shards 和 Replicas

index 中保存的数据量有可能超过单个节点的硬件限制，因此ES提供了将index划分为多个分片（shards）的功能，ES index在底层是一个或者多个物理shards的逻辑分组。创建index时，可以定义shards的数量。每个shards本身是一个功能完整的、独立的“index”，它可以保存在集群中的任何节点上。

documents 可以分布在一个index的多个shards钟，shards可以分布在集群的多个nodes上，shards使得ES的容量可以水平地扩展/分隔；通过shards的分布和并行化操作shards可以提升性能/吞吐量。

在集群中，为了提供某个shard/node故障时的故障转义机制，ES提供了replicas，它是index shards的拷贝，shards分两种——primaries和replicas。index 中的每个document 都属于一个primary shard。replica shard是primary shard的拷贝。

replicas提供了shard/node故障时的高可用，通过对repicas进行并行的查询可以扩展查询量/吞吐量。

默认情况下，ES中每个索引会被分配5个驻shards以及1各replica，即一个index有5个主shards和5个replia shards，所以一个集群至少应该有两个节点。

##### 1.6、Near Realtime（NRT）

Elasticsearch是接近实时的查询平台。意味着，从对document进行索引到它变得可以被查询会有一些延时（一般1秒）。

##### 1.7、数据检索

ES REST APIs 支持结构化查询、全文检索和将两者结合起来的复杂检索。结构化查询与 SQL 类似。全文检索会查询匹配检索字符串的所有documents并按照相关性（与检索项目的匹配程度）排序后返回。

ES 提供了JSON风格的查询语言 [Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/query-dsl.html)，也提供了[SQL](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/sql-getting-started.html)风格查询。

ES 支持聚合分析。聚合分析可以和查询一起进行。

#### 2、新建和删除 Index

新建Index，可以直接向ES服务器发出 PUT 请求
新建一个名叫weather的Index，服务器返回一个Json对象，`acknowledge`字段表示操作是否成功。

```shell
$ curl -X PUT 'localhost:9200/weather'

	{
	  "acknowledged":true,
	  "shards_acknowledged":true
	}
```
发送DELETE请求，删除Index

```shell
$ curl -X DELETE 'localhost:9200/weather'
```

#### 3、中文分词设置

首先，安装中文分词插件；（k、smartcn）

```shell
$ ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.1/elasticsearch-analysis-ik-5.5.1.zip
```

重启ES，会自动加载这个新安装的插件；

新建一个Index，指定需要分词的字段；基本上，凡是需要搜索的字段，都要单独设置一下。

```shell
$ curl -X PUT 'localhost:9200/accounts' -d 
'{
  "mappings": {
    "person": {
      "properties": {
        "user": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "desc": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        }
      }
    }
  }
}'
```

其中，`accounts`为Index，`person`为`account`的一个Type，`user`、`title`、`desc`是`person`的三个字段；`type`字段的类型'text'，`analyzer`字段文本分词器，`search_analyzer`搜索词分词器。

#### 4、数据操作

##### 4.1、新增记录

向指定的`/Index/Type`发送**PUT**请求，可以再Index里面增加一条记录。

向`/accounts/person`发送PUT请求，就可以新增一条人员记录：

```shell
$ curl -X PUT 'localhost:9200/accounts/person/1' -d '
{
  "user": "张三",
  "title": "工程师",
  "desc": "数据库管理"
}'
```

服务器返回一个 JSON 对象，会给出 Index、Type、Id、Version 等信息

```
{
  "_index":"accounts",
  "_type":"person",
  "_id":"1",
  "_version":1,
  "result":"created",
  "_shards":{"total":2,"successful":1,"failed":0},
  "created":true
}
```

在这个请求路径`/accounts/persion/1`中，其中`1`为该新增记录的id。id不一定是数据，可以指定为任意的字符串（比如“abc”等等）。

使用**POST**请求新增记录，不用指定id：

```shell
$ curl -X POST 'localhost:9200/accounts/person' -d '
{
  "user": "李四",
  "title": "工程师",
  "desc": "系统管理"
}'
```

返回json为：

```
{
  "_index":"accounts",
  "_type":"person",
  "_id":"AV3qGfrC6jMbsbXb6k1p",
  "_version":1,
  "result":"created",
  "_shards":{"total":2,"successful":1,"failed":0},
  "created":true
}
```

其中id就是一个字符串。

##### 4.2、查看记录

向`/Index/Type/Id`发出 GET 请求，就可以查看这条记录：

```shell
$ curl 'localhost:9200/accounts/person/1?pretty=true'
	{
	  "_index" : "accounts",
	  "_type" : "person",
	  "_id" : "1",
	  "_version" : 1,
	  "found" : true,
	  "_source" : {
	    "user" : "张三",
	    "title" : "工程师",
	    "desc" : "数据库管理"
	  }
	}
```

`found`字段表示查询是否成功，`_source`字段返回原始记录。

如果 Id 不正确，就查不到数据，`found`字段就是false：

```shell
$ curl 'localhost:9200/weather/beijing/abc?pretty=true'
	{
	  "_index" : "accounts",
	  "_type" : "person",
	  "_id" : "abc",
	  "found" : false
	}
```

##### 4.3、删除记录

发送DELETE请求：

```shell
$ curl -X DELETE 'localhost:9200/accounts/person/1'
```

##### 4.4、更新记录

使用PUT请求：

```shell
$ curl -X PUT 'localhost:9200/accounts/person/1' -d '
{
    "user" : "张三",
    "title" : "工程师",
    "desc" : "数据库管理，软件开发"
}'
```

返回的json对象为：

```
{
  "_index":"accounts",
  "_type":"person",
  "_id":"1",
  "_version":2,
  "result":"updated",
  "_shards":{"total":2,"successful":1,"failed":0},
  "created":false
}
```

可见，与新增记录不同，id 没有改变，但是版本（version）从 1 变成 2 ，操作结果（result）从created变成updated，created字段变成false。

使用POST请求更新记录：

```shell
curl -X POST "localhost:9200/customer/external/1/_update?pretty&pretty" -H 'Content-Type: application/json' -d'
{
  "doc": { "name": "Jane Doe", "age": 20 }
}
'
```

可以使用脚本进行更新：

```shell
curl -X POST "localhost:9200/customer/external/1/_update?pretty&pretty" -H 'Content-Type: application/json' -d'
{
  "script" : "ctx._source.age += 5"
}
'
```

`ctx._source`指的是当前的source document。

#### 5、数据查询

##### 5.1、查询所有记录

使用GET方法，直接请求`/Index/Type/_search`，就会返回所有记录

```shell
$ curl 'localhost:9200/accounts/person/_search'
	{
	  "took":2,
	  "timed_out":false,
	  "_shards":{"total":5,"successful":5,"failed":0},
	  "hits":{
	    "total":2,
	    "max_score":1.0,
	    "hits":[
	      {
	        "_index":"accounts",
	        "_type":"person",
	        "_id":"AV3qGfrC6jMbsbXb6k1p",
	        "_score":1.0,
	        "_source": {
	          "user": "李四",
	          "title": "工程师",
	          "desc": "系统管理"
	        }
	      },
	      {
	        "_index":"accounts",
	        "_type":"person",
	        "_id":"1",
	        "_score":1.0,
	        "_source": {
	          "user" : "张三",
	          "title" : "工程师",
	          "desc" : "数据库管理，软件开发"
	        }
	      }
	    ]
	  }
	}
```

返回结果的 `took`字段表示该操作的耗时（单位为毫秒），`timed_out`字段表示是否超时，`hits`字段表示命中的记录，里面子字段的含义如下：

- `total`：返回记录数，本例是2条。
- `max_score`：最高的匹配程度，本例是`1.0`。
- `hits`：返回的记录组成的数组。

返回的记录中，每条记录都有一个`_score`字段，表示匹配的程序，默认是按照这个字段降序排列。

##### 5.2、全文搜索

基于属性的检索查询，`match`查询：

```shell
$ curl 'localhost:9200/accounts/person/_search'  -d '
	{
	  "query" : { "match" : { "desc" : "管理" }},
	  "from": 1,
	  "size": 1
	}'
```

其中，`query`指定的匹配条件是`desc`字段里面包含"管理"这个词；`from`指定位移，从位置1开始（默认是从位置0开始）；`size`指定返回结果数量，默认返回10条结果。

`match`查询有一个等价的变种`match_phrase`：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_phrase": { "address": "mill lane" } }
}
'
```

查询所有结果，并按`account_number`升序排列：

```shell
curl -X GET "localhost:9200/bank/_search?q=*&sort=account_number:asc&pretty&pretty"
```

下面的`match_all`查询是等价的：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
'
```

`match_all`查询指定index的所有documents。

查询只返回两个属性：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "_source": ["account_number", "balance"]
}
'
```

##### 5.3、逻辑运算

如果有多个搜索关键字，ES认为它们是`or`的关系：

```shell
$ curl 'localhost:9200/accounts/person/_search'  -d '
	{
	  "query" : { "match" : { "desc" : "软件 系统" }}
	}'
```

`or`关系还可以表示为：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "should": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
'
```

如果要执行多个关键词的`and`搜索，必须使用布尔查询：

```shell
$ curl 'localhost:9200/accounts/person/_search'  -d '
	{
	  "query": {
	    "bool": {
	      "must": [
	        { "match": { "desc": "软件" } },
	        { "match": { "desc": "系统" } }
	      ]
	    }
	  }
	}'
```

除了`must`和`should`，还有`must`和`must_not`：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
    # 不能包含mill，也不能包含lane
      "must_not": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
'

curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
    # age必须包含40，并且state不为包含ID
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
'
```

##### 5.4、过滤

`bool`查询也支持`filter`语句。

使用`range`查询（filters的一种），查询余额在20000和30000之间的所有账户：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
'
```

并不是所有的查询都会计算产生`_score`，`filter`语句不会影响`_score`，仅仅过滤的时候不会计算`_score`。

##### 5.5、聚合

在Elasticsearch中，可以执行查询返回`hits`，同时在一个相应中还可以返回`hits`聚合的结果。可以运行查询和多个聚合并且一次请求响应获取两者的结果。

如下，为 `terms` 聚合的例子，按照`state` 分组，返回数量最多的 10 个（默认值）`state`：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
'
```

其中`size=0`的作用是不显示`hits`，只显示聚合结果，否则将显式 `hits`。

响应示例：

```
{
  "took": 29,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits" : {
    "total" : 1000,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "group_by_state" : {
      "doc_count_error_upper_bound": 20,
      "sum_other_doc_count": 770,
      "buckets" : [ {
        "key" : "ID",
        "doc_count" : 27
      }, {
        "key" : "TN",
        "doc_count" : 23
      },
      ...
      ...
      , {
        "key" : "MO",
        "doc_count" : 20
      } ]
    }
  }
}
```

可见`state`为`ID`的`account`数为27……

在`terms` 聚合内部，还可以嵌套 `avg`聚合，计算每个`state`分组内的余额的平均值：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
'
```

其中聚合`average_balance`嵌入在`group_by_state`聚合中。这是所有聚合的通用模式。

还可以进一步使用余额平均值进行排序：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
'
```

另外，还可以根据分桶进行聚合。比如，按照年纪分桶（brackets ，20-29，30-39，40-49）、然后按照性别进行分组，最后获取每个年龄段、每个性别的平均账户余额：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender.keyword"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}
'
```

#### 6、批处理

使用`_bulk` API 可以批量进行 documents 的增（index）删（delete）改（update）。

新增两条记录：

```shell
curl -X POST "localhost:9200/customer/external/_bulk?pretty&pretty" -H 'Content-Type: application/json' -d'
{"index":{"_id":"1"}}
{"name": "John Doe" }
{"index":{"_id":"2"}}
{"name": "Jane Doe" }
'
```

一个bulk操作中同时进行删除和更新：

```shell
curl -X POST "localhost:9200/customer/external/_bulk?pretty&pretty" -H 'Content-Type: application/json' -d'
{"update":{"_id":"1"}}
{"doc": { "name": "John Doe becomes Jane Doe" } }
{"delete":{"_id":"2"}}
'
```

bulk操作中部分失败，不会导致整个处理失败，通过返回状态可以查看执行的结果。

导入`accounts.json` 文件中的数据：

curl -H "Content-Type: application/json" -XPOST "localhost:9200/bank/_doc/_bulk?pretty&refresh" --data-binary "@accounts.json"