### Java API

Elasticsearch的客户端种类比较多，有Java 客户端，还有Java REST客户端（包含 Java 底层 REST 客户端和Java 高层 REST 客户端），还有第三方`io.searchbox`的`Jest` API。貌似，最新的版本（7.12）已经废弃了Java API转向支持高层的Java REST客户端（执行 HTTP 请求，而不是序列化的 Java 请求），未来的 8.0 版本会完全移除 Java API。

[Java API ](https://www.elastic.co/guide/en/elasticsearch/client/java-api/6.8/java-api.html)操作通过 Client 对象执行，所有的操作都是异步的。