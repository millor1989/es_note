### Document APIs

单document APIs：

- Index API
- Get API
- Delete API
- Update API

多 document APIs：

- Multi Get API
- Bulk API
- Delete By Query API
- Update By Query API
- Reindex API

#### 1、documents 读写

ES 中的 index 是被划分为 shards 的，每个 shard可以有多个拷贝。这些拷贝被称为 replication group，在documents 增加或者删除时必须保持同步。保持shard 拷贝同步并服务于读取得进程被称为 data replication model。

ES 的 data replication model 是基于 primary-backup model的。这个模型基于将 replication group 中的一个拷贝作为 primary shard。其它拷贝叫作 replica shards。primary shard 作为所有 indexing 操作的主入口点。它负责验证操作，并保证操作正确。一旦 primary 接受了一个index 操作，它也将负责对其它拷贝重复这个操作。

##### 1.1、基本的写模型

ES 的indexing操作首先会通过路由（routing，一般基于 document ID）指向一个 replication group。确定了replication group 之后，在内部，操作会指向group的当前primary shard。primary shard 负责验证操作并将操作应用到其他replicas。replicas 可以离线，primary 不必对所有的replicas重复操作。ES 的master node 维持着一个应该接收操作的 shard 拷贝——in-sync 拷贝。primary 只用对 in-sync拷贝重复操作。

primary shard 的基本流程：

1. 验证操作，如果操作在结构上是非法（比如，number 属性的值是 object）的则拒绝操作；
2. 在本地执行操作，即indexing 或者删除相关的 document。这一步，也会验证属性的内容，并且在不合法时拒绝（比如，keyword 的值太长）
3. 对当前 in-sync 拷贝集合中的每个 replica 重复操作。如果有多个，则并行进行。
4. 如果所有的 replicas 都完成了操作并向 primary 发出响应，primary 通知客户端请求成功。

**1.1.1、故障处理**

primary 故障时，primary 所在 node 通知 master。indexing 操作等待（默认 1 分钟）master 从 replicas 中提拔新的 primary，然后由新的 primary 执行处理。另外，master 也会监控 nodes 的健康状况，并可能撤销 primary（一般，在 primary 所在node 发生网络异常时发生）。

primary 完成操作后，向replica 重复操作的过程中 replica 故障或者网络故障时，primary 通知 master 将未能完成操作的 replica从 in-sync replica 集合中移除，master 通知 primary 移除完成后，primary 报告操作成功。master 会通知其他 node 启动一个新的 shard 拷贝以将系统恢复到健康状态。

##### 1.2、基本读模型

primary-backup 模型的好处是，所有的 shard 拷贝都是一致的。因此，只有一个 in-sync 拷贝也足以响应读取请求。

node 接收到读请求时，这个 node负责将请求发送到持有相关 shards 的 nodes，生成响应，并对client 做出响应。收到请求的 node 被称为 coordinating node（协调 node）。基本流程为：

1. 将读操作之前相关的shards。注意，由于大多数的查询都会被发送到一个或者多个 indicies，它们一般需要从多个 shards读取数据，每个shard 代表数据的一个不同子集。
2. 从 shard replication group 选取每个相关 shard （primary 或者 replica）的活跃拷贝。默认情况下，ES 只会简单地在 shard 拷贝之间循环。
3. 将 shard 级别的读取请求发送到选定的拷贝。
4. 组合结果并做出响应。注意，对于 Get By Id 查询，只会有一个相关的 shard，这一步回跳过。

##### 1.3、一些简单地含义

由于读写可以并发执行，这两个操作会互相影响。内在含义：

- 正常情况下，每个读操作只会对每个相关 replication group 执行一次。

- primary 首先本地执行indexing，然后重复请求。对于并发读操作来说可能在收到变化通知前已经看到了变化。
- 只有两个数据拷贝这个模型就能容错。而quorum-based 系统实现容错的最小拷贝数量是 3。