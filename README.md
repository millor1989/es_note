### ElasticSearch

文件索引和全文搜索引擎，可以快速地储存、搜索和分析海量数据。

ElasticSearch是Lucene的封装，用Java开发的，提供了REST API的操作接口，开箱即用。

Elastic 默认运行在9200端口，直接访问该端口会得到 Elastic 的信息（包括节点、集群、版本等信息）。

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

默认情况下，Elastic只允许本机访问，如果需要远程访问，修改Elastic安装目录的`config/ealsticsearch.yml`设置“network.host:0.0.0.0”，重启Elastic。

#### 1、基本概念

##### 1.1、Node、Cluster

Elastic本质上是一个**分布式数据库**，允许多台服务器协同工作，每台服务器可运行多个ES实例。
一个ES实例为一个**节点**（**node**），一组节点构成一个**集群**（**cluster**）

##### 1.2、Index

ES会**索引所有字段**，经过处理后写入一个反向索引（Inverted Index）。查找数据的时候，直接查找该索引。	
索引（Index）是ES数据管理的顶层单位。它是单个数据库的同义词。每个**Index（数据库）**的名字必须是小写。
查看当前节点的所有Index	

```shell
$ curl -X GET 'http://localhost:9200/_cat/indices?v'
```

##### 1.3、Document

Index里面的**单条记录**称为Document（文档）。许多条Document构成了一个Index。Document使用Json格式表示，同一个Index里面的Document，不要求有相同的结构（schema）；但是如果有相同的结构，有利于提高搜索效率。		

##### 1.4、Type

Document可以分组，这种**分组**就叫做Type；它是虚拟的逻辑分组，用来过滤Document。不同的Type应该有相似的结构（schema），性质完全不同的数据应该存在两个Index，而不是一个Index的两个Type。
查看每个Index所包含的Type

```shell
$ curl 'localhost:9200/_mapping?pretty=true'
```

根据规划，Elastic 6.x的版本只允许每个Index包含一个Type，* 7.X 的版本将会彻底移除 Type *

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

使用**POST**请求新增记录：

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

##### 5.2、全文搜索

```shell
$ curl 'localhost:9200/accounts/person/_search'  -d '
	{
	  "query" : { "match" : { "desc" : "管理" }},
	  "from": 1,
	  "size": 1
	}'
```

其中，`query`指定的匹配条件是`desc`字段里面包含"管理"这个词；`from`指定位移，从位置1开始（默认是从位置0开始）；`size`指定返回结果数量，默认返回10条结果。

##### 5.3、逻辑运算

如果有多个搜索关键字，ES认为它们是`or`的关系：

```shell
$ curl 'localhost:9200/accounts/person/_search'  -d '
	{
	  "query" : { "match" : { "desc" : "软件 系统" }}
	}'
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

