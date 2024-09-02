### API约定

ES REST APIs 是使用 JSON 通过 HTTP 执行的。

#### 1、多个Indices

使用`index`参数的大多数APIs都支持跨多个indices执行，比如`test1,test2,test3`表示（或者`_all`来表示所有的indices）。还支持通配符，比如`test*`、`*test`、`te*t`或者`*test*`，并且还能增（`+`）和减（`-`），比如`+test*,-test3`。

所有的多indices API都支持如下的url查询字符串参数：

- `ignore_unavailable`：是否忽略指定的indices中不可用的indices（indices不可用，包括indices不存在或者indices已经被关闭）。

- `allow_no_indices`：如果通配符匹配不到具体的indices，是否失败。当使用`_all`，`*`或者没有指定index时，这个设置也起作用。当别名指向关闭的index时，这个设置也应用于别名（alias）。

- `expand_wildcards`：控制通配符indices表达式扩展到哪种具体的indices。如果为`open`，通配符表达式扩展到打开的indices，如果为`closed`则扩展到关闭的indices。还可以同时指定为`open,closed`。

  如果为`none`，通配符扩展会关闭，如果指定为`all`，则等价于 `open,closed`。

#### 2、index名称的Date math支持

通过date math index名称解析，能够查询一个时间范围的indices，从而不用查询所有时间的indices并过滤结果。限制查询indices的数量还能降低集群的负载并提升查询执行性能。

date math index的格式：

```shell
<static_name{date_math_expr{date_format|time_zone}}>
```

其中：

| `staic_name`     | 名称的静态文本部分                 |
| ---------------- | ---------------------------------- |
| `date_math_expr` | 动态计算日期的动态date math表达式  |
| `date_format`    | 计算日期使用格式。默认`YYYY.MM.dd` |
| `time_zone`      | 时区。默认`utc`                    |

date math index名称表达式必须用尖括号包括，所有指定的字符串应该进行URI编码，比如：

```shell
# GET /<logstash-{now/d}>/_search
curl -X GET "localhost:9200/%3Clogstash-%7Bnow%2Fd%7D%3E/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query" : {
    "match": {
      "test": "data"
    }
  }
}'
```

假如当前时间是UTC 2024年3月22日中午，表达式和对应的结果为：

| Expression                              | Resolves to           |
| --------------------------------------- | --------------------- |
| `<logstash-{now/d}>`                    | `logstash-2024.03.22` |
| `<logstash-{now/M}>`                    | `logstash-2024.03.01` |
| `<logstash-{now/M{YYYY.MM}}>`           | `logstash-2024.03`    |
| `<logstash-{now/M-1M{YYYY.MM}}>`        | `logstash-2024.02`    |
| `<logstash-{now/d{YYYY.MM.dd|+12:00}}>` | `logstash-2024.03.23` |

如果index名称模板的静态部分包含了字符`{`和`}`，需要使用反斜杠`\`进行转义：

- `<elastic\{ON\}-{now/M}>` 解析为 `elastic{ON}-2024.03.01`

查询最近3天的Logstash：

```shell
# GET /<logstash-{now/d-2d}>,<logstash-{now/d-1d}>,<logstash-{now/d}>/_search
curl -X GET "localhost:9200/%3Clogstash-%7Bnow%2Fd-2d%7D%3E%2C%3Clogstash-%7Bnow%2Fd-1d%7D%3E%2C%3Clogstash-%7Bnow%2Fd%7D%3E/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query" : {
    "match": {
      "test": "data"
    }
  }
}'

```

#### 3、通用选项

可以用到所有REST APIs的选项

##### 3.1、美化结果

当请求追加了`?pretty=true`，返回的JSON会被美化（仅用于debug）。为请求追加`?format=yaml`返回的结果是更具可读性的yaml格式。

##### 3.2、人类可读输出

返回适合人类阅读格式的统计数据，(比如 `"exists_time": "1h"` or `"size": "1kb"`) 或者适合机器阅读的格式 (比如 `"exists_time_in_millis": 3600000` or `"size_in_bytes": 1024`)。`?human=true`返回适合人读的格式，默认是适合机器的。

##### 3.3、date math

接收格式化日期值的大多数参数，比如`range`查询中的`gt`和`lt`，或者`daterange`聚合中的`from`和`to`——都可以使用date maths。

date math表达式以一个锚日期开始（可以是`now`，也可以是以`||`结束的日期字符串）。锚日期可以跟一个或多个数学表达式：

- `+1h` - 加1小时
- `-1d` - 减一天
- `/d` - 截取日期部分

支持的日期单位：

| `y`  | years   |
| ---- | ------- |
| `M`  | months  |
| `w`  | weeks   |
| `d`  | days    |
| `h`  | hours   |
| `H`  | hours   |
| `m`  | minutes |
| `s`  | seconds |

假如`now`是`2001-01-01 12:00:00`：

**`now+1h`**——毫秒的`now` 加一个小时。结果是: `2001-01-01 13:00:00`

**`now-1h`**——毫秒的`now` 减一个小时，结果是: `2001-01-01 11:00:00`

**`now-1h/d`**——毫秒的`now` 截取日期部分，结果是: `2001-01-01 00:00:00`

**`2001-01-01\|\|+1M/d`**——毫秒的`2001-01-01` 加一个月。结果是: `2001-02-01 00:00:00`

##### 3.4、响应过滤

所有的REST APIs可以通过`filter_path`参数过滤返回的响应。

```shell
curl -X GET "localhost:9200/_search?q=elasticsearch&filter_path=took,hits.hits._id,hits.hits._score&pretty"

```

结果：

```
{
  "took" : 3,
  "hits" : {
    "hits" : [
      {
        "_id" : "0",
        "_score" : 1.6375021
      }
    ]
  }
}
```

此外，`*`通配符匹配任何属性或者属性名的一部分；`**`通配符直接匹配任意属性；`-`字符排除一个或多个属性；同一个表达式中可以既包括排除过滤又包括包含过滤。

```
curl -X GET "localhost:9200/_cluster/state?filter_path=metadata.indices.*.state,-metadata.indices.logstash-*&pretty"

```

这个例子中，会首先应用排除过滤，然后应用包含过滤，结果为：

```
{
  "metadata" : {
    "indices" : {
      "index-1" : {"state" : "open"},
      "index-2" : {"state" : "open"},
      "index-3" : {"state" : "open"}
    }
  }
}
```

es有时会直接返回一个属性的原始值，比如`_source`属性。如果要过滤`_source`的属性，应该使用`filter_path`参数结合已经存在的`_source`参数进行过滤：

```shell
curl -X POST "localhost:9200/library/book?refresh&pretty" -H 'Content-Type: application/json' -d'
{"title": "Book #1", "rating": 200.1}
'
curl -X POST "localhost:9200/library/book?refresh&pretty" -H 'Content-Type: application/json' -d'
{"title": "Book #2", "rating": 1.7}
'
curl -X POST "localhost:9200/library/book?refresh&pretty" -H 'Content-Type: application/json' -d'
{"title": "Book #3", "rating": 0.1}
'
curl -X GET "localhost:9200/_search?filter_path=hits.hits._source&_source=title&sort=rating:desc&pretty"

```

结果为：

```
{
  "hits" : {
    "hits" : [ {
      "_source":{"title":"Book #1"}
    }, {
      "_source":{"title":"Book #2"}
    }, {
      "_source":{"title":"Book #3"}
    } ]
  }
}
```

##### 3.5、Flat设置

`flat_settings`，为true时，将结果扁平化：

```shell
curl -X GET "localhost:9200/twitter/_settings?flat_settings=true&pretty"
```

结果：

```
{
  "twitter" : {
    "settings": {
      "index.number_of_replicas": "1",
      "index.number_of_shards": "1",
      "index.creation_date": "1474389951325",
      "index.uuid": "n6gzFZTgS664GUfx0Xrpjw",
      "index.version.created": ...,
      "index.provided_name" : "twitter"
    }
  }
}
```

##### 3.6、参数

REST参数遵循下划线写法。

##### 3.7、Boolean值

所有的REST APIs支持通过`false`、`0`、`no`、`off`提供布尔Flase值。其它值都会被当作True值。

5.3.0版本之后废弃了仅仅支持`true`或`false`。

##### 3.8、数字值

在支持JSON数字类型的基础上，支持`string`数值。

##### 3.9、时间单位

使用时长时，必须指定单位。支持的单位是：

| `d`      | days         |
| -------- | ------------ |
| `h`      | hours        |
| `m`      | minutes      |
| `s`      | seconds      |
| `ms`     | milliseconds |
| `micros` | microseconds |
| `nanos`  | nanoseconds  |

##### 3.10、字节大小单位

##### 3.11、无单位数量

##### 3.12、距离单位

##### 3.13、模糊性（fuzziness）

某些查询和APIs使用`fuzziness` 参数，支持参数进行非精确的模糊匹配。

当查询`text`或者`keyword`属性时，`fuzziness` 被解释为Levenshtein Edit Distance——让一个字符串变得与另一个字符串相同时需要一个字符变化的数量。

`fuzziness` 参数可以指定为：

- `0`，`1`，`2`：允许的最大Levenshtein Edit Distance

- `AUTO`：基于词的长度生成一个edit distance。对于长度：

  `0..2`：必须精确地匹配

  `3..5`：允许1个edit distance

  `>5`：允许2个edit distance

  `AUTO`是推荐的`fuzziness` 值。

##### 3.14、开启栈追踪

`error_trace=true`，查看查询错误的栈信息。

##### 3.15、查询字符串中的请求体

对于不接收请求体的非 POST 请求 API，可以将请求体作为 `source` 查询参数的字符串值。这样用的时候，`source_content_type` 参数应该设置为与 `source` 参数包含内容对应的 media type，比如 `appliation/json`。

##### 3.16、Content-Type自动检测

5.3版本之后废弃了必须提供合适的Content-Type header

#### 4、基于URL的访问控制

许多用户使用代理和基于URL的访问控制来安全的访问Elasticsearch indices。对于`multi-search`、`multi-get`和`bulk`请求，用户可以选择在URL中指定一个index 并在请求体中的单个请求指定index。这会使基于URL的访问控制受到挑战。

为了防止用户覆盖URL中指定的index，可以在`elasticsearch.yml`文件中增加设置：

```
rest.action.multi.allow_explicit_index: false
```

默认值true。设置为false后，es会拒绝在请求体中明确地指定index的请求。

