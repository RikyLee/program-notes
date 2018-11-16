## Elasticsearch 基础

### 安装并运行Elasticserch

- 先安装Java8

- [官网](https://www.elastic.co/downloads/elasticsearch)下载elasticsearch二进制包

- 解压文件到指定位置，并启动

  ```shell
    tar xf elasticsearch-<version>.tgz
    cd elasticsearch-<version>
    ./bin/elasticsearch
  ```

  如果你想把 Elasticsearch 作为一个守护进程在后台运行，那么可以在后面添加参数 -d

- 测试 Elasticsearch 是否启动成功，可以打开另一个终端，执行以下操作`curl 'http://localhost:9200/?pretty'`

  ```json
  {
    name: "2kiqbv1",
    cluster_name: "es-olami-logger",
    cluster_uuid: "v-XITjF4Q4C0cXlojVEE7A",
    version: {
      number: "6.4.0",
      build_flavor: "default",
      build_type: "tar",
      build_hash: "595516e",
      build_date: "2018-08-17T23:18:47.308994Z",
      build_snapshot: false,
      lucene_version: "7.4.0",
      minimum_wire_compatibility_version: "5.6.0",
      minimum_index_compatibility_version: "5.0.0"
    },
    tagline: "You Know, for Search"
  }
  ```

### 和Elasticsearch交互

- JAVA API Elasticsearch内置两个客户端Node client 和 Transport client

  - Node client 节点客户端作为一个非数据节点加入到本地集群中。换句话说，它本身不保存任何数据，但是它知道数据在集群中的哪个节点中，并且可以把请求转发到正确的节点

  - Transport client 轻量级的传输客户端可以将请求发送到远程集群。它本身不加入集群，但是它可以将请求转发到集群中的一个节点上

  两个Java客户端都是通过9300端口并使用Elasticsearch的原生传输协议和集群交互。集群中的节点通过端口 9300彼此通信。如果这个端口没有打开，节点将无法形成一个集群。

- RESTful API 所有其他语言可以使用 RESTful API 通过端口 9200 和 Elasticsearch 进行通信

  一个 Elasticsearch 请求和任何 HTTP 请求一样由若干相同的部件组成：

  ```shell
    curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
  ```

  - VERB HTTP方法：`GET`、 `POST`、 `PUT`、 `HEAD`、`DELETE`
  - PROTOCOL http 或者 https`
  - HOST Elasticsearch 集群中任意节点的主机名，或者用 localhost 代表本地机器上的节点
  - PORT 运行Elasticsearch HTTP服务的端口号，默认是 9200
  - PATH PI 的终端路径
  - QUERY_STRING 任意可选的查询字符串参数
  - BODY 一个 JSON 格式的请求体 (如果请求需要的话)

  ```shell
    curl -XGET 'http://localhost:9200/_count?pretty' -d '
    {
        "query": {
            "match_all": {}
        }
    }
  ```

Elasticsearch不仅存储文档，而且索引每个文档的内容使之可以被检索。

### 分布式特性

Elasticsearch 可以横向扩展至数百（甚至数千）的服务器节点，同时可以处理PB级数据,Elasticsearch 尽可能地屏蔽了分布式系统的复杂性，列举了一些在后台自动执行的操作：

- 分配文档到不同的容器或分片中，文档可以储存在一个或多个节点中
- 按集群节点来均衡分配这些分片，从而对索引和搜索过程进行负载均衡
- 复制每个分片以支持数据冗余，从而防止硬件故障导致的数据丢失
- 将集群中任一节点的请求路由到存有相关数据的节点
- 集群扩容时无缝整合新节点，重新分配分片以便从离群节点恢复

### 分布式存储

- 路由一个文档到一个分片

  ```text
  shard = hash(routing) % number_of_primary_shards
  ```

  routing 是一个可变值，默认是文档的 _id ，也可以设置成一个自定义的值。 routing 通过 hash 函数生成一个数字，然后这个数字再除以 number_of_primary_shards （主分片的数量）后得到 余数 。这个分布在 0 到 number_of_primary_shards-1 之间的余数，就是我们所寻求的文档所在分片的位置，所有的文档 API（ get 、 index 、 delete 、 bulk 、 update 以及 mget ）都接受一个叫做 routing 的路由参数 ，通过这个参数我们可以自定义文档到分片的映射。

- 一致性（consistency）

  在默认设置下，即使仅仅是在试图执行一个_写_操作之前，主分片都会要求 必须要有 _规定数量(quorum)_（或者换种说法，也即必须要有大多数）的分片副本处于活跃可用状态，才会去执行_写_操作(其中分片副本可以是主分片或者副本分片)。这是为了避免在发生网络分区故障（network partition）的时候进行_写_操作，进而导致数据不一致，_规定数量_即：

    ```text
    int( (primary + number_of_replicas) / 2 ) + 1
    ```
  consistency 参数的值可以设为 one （只要主分片状态 ok 就允许执行_写_操作）,all（必须要主分片和所有副本分片的状态没问题才允许执行_写_操作）, 或 quorum 。默认值为 quorum , 即大多数的分片副本状态没问题就允许执行_写_操作。

  timeout参数 如果没有足够的副本分片，Elasticsearch会等待，希望更多的分片出现。默认情况下，它最多等待1分钟。 如果你需要，你可以使用 timeout 参数 使它更早终止： 100 100毫秒，30s 是30秒

### 搜索

- 空查询

  搜索API的最基础的形式是没有指定任何查询的空搜索 ，它简单地返回集群中所有索引下的所有文档：

  ```text
  GET /_search
  ```

  返回的结果,类似如下结构：

  ```json
  {
    "hits" : {
        "total" :       14,
        "hits" : [
          {
            "_index":   "us",
            "_type":    "tweet",
            "_id":      "7",
            "_score":   1,
            "_source": {
              "date":    "2014-09-17",
              "name":    "John Smith",
              "tweet":   "The Query DSL is really powerful and flexible",
              "user_id": 2
            }
        },
          ... 9 RESULTS REMOVED ...
        ],
        "max_score" :   1
    },
    "took" :           4,
    "_shards" : {
        "failed" :      0,
        "successful" :  10,
        "total" :       10
    },
    "timed_out" :      false
  }
  ```

  hits: 返回结果中最重要的部分是 hits ，它包含total字段来表示匹配到的文档总数，并且一个 hits 数组包含所查询结果的前十个文档。hits 数组中每个结果包含文档的 _index 、 _type 、 _id ，加上 _source 字段。每个结果还有一个 _score ，它衡量了文档与查询的匹配程度。默认情况下，首先返回最相关的文档结果，就是说，返回的文档是按照 _score 降序排列的。

  took: 告诉我们执行整个搜索请求耗费了多少毫秒。

  _shards: 告诉我们在查询中参与分片的总数，以及这些分片成功了多少个失败了多少个。

  timed_out: 值告诉我们查询是否超时，默认情况下，搜索请求不会超时。 如果低响应时间比完成结果更重要，你可以指定 timeout 为 10 或者 10ms（10毫秒），或者 1s（1秒）：

  ```text
  GET /_search?timeout=10ms
  ```

  在请求超时之前，Elasticsearch 将会返回已经成功从每个分片获取的结果。

- 多类型，多索引

  上述返回的结果是有不同类型的文档和不同的索引聚合而成的，如果不对某一特殊的索引或者类型做限制，就会搜索集群中的所有文档。Elasticsearch 转发搜索请求到每一个主分片或者副本分片，汇集查询出的前10个结果，并且返回给我们。

  你想在一个或多个特殊的索引并且在一个或者多个特殊的类型中进行搜索。我们可以通过在URL中指定特殊的索引和类型达到这种效果，如下所示：

    ```text
    /_search 在所有的索引中搜索所有的类型
    /gb/_search 在 gb 索引中搜索所有的类型
    /gb,us/_search 在 gb 和 us 索引中搜索所有的文档
    /g*,u*/_search 在任何以 g 或者 u 开头的索引中搜索所有的类型
    /gb/user/_search 在 gb 索引中搜索 user 类型
    /gb,us/user,tweet/_search 在 gb 和 us 索引中搜索 user 和 tweet 类型
    /_all/user,tweet/_search 在所有的索引中搜索 user 和 tweet 类型
    ```

- 分页

  和 SQL 使用 LIMIT 关键字返回单个 page 结果的方法相同，Elasticsearch 接受 from 和 size 参数：
    ```text
    size 显示应该返回的结果数量，默认是 10
    from 显示应该跳过的初始结果数量，默认是 0
    ```
  在分布式系统中深度分页，我们可以假设在一个有 5 个主分片的索引中搜索。 当我们请求结果的第一页（结果从 1 到 10 ），每一个分片产生前 10 的结果，并且返回给 协调节点 ，协调节点对 50 个结果排序得到全部结果的前 10 个。

  现在假设我们请求第 1000 页--结果从 10001 到 10010 。所有都以相同的方式工作除了每个分片不得不产生前10010个结果以外。 然后协调节点对全部 50050 个结果排序最后丢弃掉这些结果中的 50040 个结果。

  可以看到，在分布式系统中，对结果排序的成本随分页的深度成指数上升。这就是 web 搜索引擎对任何查询都不要返回超过 1000 个结果的原因。

- 轻量搜索

## Elasticsearch 集群

### 集群的概念

一个运行中的 Elasticsearch 实例称为一个 节点，而集群是由一个或者多个拥有相同 cluster.name 配置的节点组成， 它们共同承担数据和负载的压力。当有节点加入集群中或者从集群中移除节点时，集群将会重新平均分布所有的数据。

当一个节点被选举成为主节点时,它将负责管理集群范围内的所有变更，例如增加、删除索引，或者增加、删除节点等。而主节点并不需要涉及到文档级别的变更和搜索等操作，所以当集群只拥有一个主节点的情况下，即使流量的增加它也不会成为瓶颈。

我们可以将请求发送到集群中的任何节点 ，包括主节点。每个节点都知道任意文档所处的位置，并且能够将我们的请求直接转发到存储我们所需文档的节点。无论我们将请求发送到哪个节点，它都能负责从各个包含我们所需文档的节点收集回数据，并将最终结果返回給客户端。 Elasticsearch对这一切的管理都是透明的。

### 集群的监控

- GET /_cluster/health

```json
{
  cluster_name: "es-olami-logger",
  status: "yellow",
  timed_out: false,
  number_of_nodes: 1,
  number_of_data_nodes: 1,
  active_primary_shards: 27,
  active_shards: 27,
  relocating_shards: 0,
  initializing_shards: 0,
  unassigned_shards: 10,
  delayed_unassigned_shards: 0,
  number_of_pending_tasks: 0,
  number_of_in_flight_fetch: 0,
  task_max_waiting_in_queue_millis: 0,
  active_shards_percent_as_number: 72.97297297297297
}
```

status 字段指示着当前集群在总体上是否工作正常。它的三种颜色含义如下:

```text
green:所有的主分片和副本分片都正常运行

yellow:所有的主分片都正常运行，但不是所有的副本分片都正常运行

red:有主分片没能正常运行
```

unassigned_shards 字段表示未分配的副本分片个数，status为yellow状态下，硬件故障时有丢失数据的风险

### 索引

索引保存相关数据，实际上是指向一个或者多个物理分片的逻辑命名空间。

一个分片是一个底层的工作单元，它仅保存了全部数据中的一部分。一个分片是一个Lucene的实例，它本身就是一个完整的搜索引擎。我们的文档被存储和索引到分片内，但是应用程序是直接与索引而不是与分片进行交互。Elasticsearch利用分片将数据分发到集群内各处。分片是数据的容器，文档保存在分片内，分片又被分配到集群内的各个节点里。当你的集群规模变更时，Elasticsearch会自动的在各节点中迁移分片，使得数据仍然均匀分布在集群里。

索引内任意一个文档都归属于一个主分片，所以主分片的数目决定着索引能够保存的最大数据量，一个主分片最大能够存储Integer.MAX_VALUE-128个文档

一个副本分片只是一个主分片的拷贝，副本分片作为硬件故障时保护数据不丢失的冗余备份，并为搜索和返回文档等读操作提供服务。

- 索引文档的创建

  一个文档不仅仅包含它的数据 ，也包含元数据信息。 三个必须的元数据元素如下：

  ```text
  _index:文档在哪存放，即文档的索引名称
  _type:文档表示的对象类别
  _id:文档唯一标识，可以自己指定，也可以有ES自动生成，与_type配合使用可以唯一确定 Elasticsearch 中的一个文档
  ```

  - 自定义ID

    ```text
    PUT  /{index}/{type}/{id}
    {
      "field": "value",
      ...
    }
    ```
  - 自动生成ID

    ```text
      POST  /{index}/{type}
      {
        "field": "value",
        ...
      }
    ```
    自动生成的 ID 是 URL-safe、 基于 Base64 编码且长度为20个字符的 GUID 字符串。 这些 GUID 字符串由可修改的 FlakeID 模式生成，这种模式允许多个节点并行生成唯一 ID ，且互相之间的冲突概率几乎为零。_index 、 _type 和 _id 的组合可以唯一标识一个文档,所以，确保创建一个新文档的最简单办法是，使用索引请求的 POST 形式让 Elasticsearch 自动生成唯一 _id，然而，如果已经有自己的 _id ，那么我们必须告诉 Elasticsearch ，只有在相同的 _index 、 _type 和 _id 不存在时才接受我们的索引请求。有两种实现方式：

    ```text
    PUT /{index}/{type}/{id}?op_type=create
    { ... }

    PUT /{index}/{type}/{id}/_create
    { ... }

    ```

    如果创建新文档的请求成功执行，Elasticsearch 会返回元数据和一个 201 Created的HTTP响应码。另一方面，如果具有相同的_index、_type和_id的文档已经存在，Elasticsearch将会返回409 Conflict响应码
    ```json
    {
      "error": {
          "root_cause": [
            {
                "type": "document_already_exists_exception",
                "reason": "[type][id]: document already exists",
                "shard": "0",
                "index": "index"
            }
          ],
          "type": "document_already_exists_exception",
          "reason": "[type][id]: document already exists",
          "shard": "0",
          "index": "index"
      },
      "status": 409
    }
    ```

- 索引的查询

  - 取回所有文档

    ```text
    GET /{index}/{type}/_search
    {
      "query":{
        "match_all":{}
      }
    }
    ```
    返回指定索引、类型下的最新的10条数据

  - 取回指定ID的文档

    ```text
    GET /{index}/{type}/{id}
    ```

  - 取回指定ID的文档的部分字段

    ```text
    GET /{index}/{type}/{id}?_source=filed1,filed2
    ```

  - 取回指定ID的文档的_source内容

    ```text
    GET /{index}/{type}/{id}/_source
    ```

  - 查询指定文档存不存在

    ```text
    HEAD /{index}/{type}/{id}
    ```

    如果返回的是200 - OK表示文档存在

  - 取回多个文档

    可以使用 multi-get 或者 mget API 来将这些检索请求放在一个请求中，将比逐个文档请求更快地检索到全部文档。

    mget API 要求有一个 docs 数组作为参数，每个元素包含需要检索文档的元数据，包括_index_type和_id 。如果你想检索一个或者多个特定的字段，那么你可以通过_source参数来指定这些字段的名字：
    ```text
    GET /_mget
    {
      "docs" : [
          {
            "_index" : "index1",
            "_type" :  "type1",
            "_id" :    2
          },
          {
            "_index" : "index2",
            "_type" :  "type2",
            "_id" :    1,
            "_source": "filed"
          }
      ]
    }
    ```
    如果想检索的数据都在相同的 _index 中（甚至相同的 _type 中），则可以在 URL 中指定默认的 /_index 或者默认的 /_index/_type:
    ```text
    GET /index/type/_mget
    {
      "docs" : [
          { "_id" : 2 },
          { "_type" : "type1", "_id" :   1 }
      ]
    }
    ```
    事实上，如果所有文档的 _index 和 _type 都是相同的，你可以只传一个 ids 数组，而不是整个 docs 数组：
    ```text
    GET /index/type/_mget
    {
      "ids" : [ "2", "1" ]
    }
    ```

  可以在url最后拼接?pretty，使得JSON响应体可读性增强。返回结果中包含found字段，值为true时表示查询到结果，false时表示无结果。

- 索引的修改

  在 Elasticsearch 中文档是不可改变的，不能修改它们。如果想要更新现有的文档，需要重建索引或者进行替换， 我们可以使用相同的index API 进行实现

  ```text
    PUT  /{index}/{type}/{id}
    {
      "field": "value",
      ...
    }
    ```
  实际上在内部，Elasticsearch 已将旧文档标记为已删除，并增加一个全新的文档,尽管你不能再对旧版本的文档进行访问，但它并不会立即消失。如果使用update API 将执行吐下步骤：

    ```text
    1.从旧文档构建 JSON
    2.更改该JSON
    3.删除旧文档
    4.索引一个新文档
    ```
  update API接受routing 、 replication 、 consistency 和 timeout 参数。

- 索引的删除

  删除文档的语法和我们所知道的规则相同，只是使用DELETE方法：

    ```text
    DELETE /{index}/{type}/{id}
    ```

  如果找到该文档，Elasticsearch 将要返回一个200 ok的HTTP响应码，和一个类似以下结构的响应体。注意，字段_version值已经增加:

    ```json
    {
      "found" :    true,
      "_index" :   "index",
      "_type" :    "type",
      "_id" :      "id",
      "_version" : 3
    }
    ```
  如果文档没有 找到，我们将得到 404 Not Found 的响应码和类似这样的响应体：
    ```json
    {
      "found" :    false,
      "_index" :   "index",
      "_type" :    "type",
      "_id" :      "id",
      "_version" : 3
    }
    ```
  即使文档不存在（ Found 是 false ）， _version 值仍然会增加。这是 Elasticsearch 内部记录本的一部分，用来确保这些改变在跨多节点时以正确的顺序执行。删除文档不会立即将文档从磁盘中删除，只是将文档标记为已删除状态。随着你不断的索引更多的数据，Elasticsearch 将会在后台清理标记为已删除的文档。

- 批量操作索引

  bulk API 允许在单个步骤中进行多次 create 、 index 、 update 或 delete 请求。 如果你需要索引一个数据流比如日志事件，它可以排队和索引数百或数千批次。bulk 与其他的请求体格式稍有不同，如下所示：
    ```text
    { action: { metadata }}\n
    { request body        }\n
    { action: { metadata }}\n
    { request body        }\n
    ...
    ```

    一个完整的 bulk 请求 有以下形式:

    ```text
    POST /_bulk
    { "delete": { "_index": "website", "_type": "blog", "_id": "123" }}
    { "create": { "_index": "website", "_type": "blog", "_id": "123" }}
    { "title":    "My first blog post" }
    { "index":  { "_index": "website", "_type": "blog" }}
    { "title":    "My second blog post" }
    { "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" : 3} }
    { "doc" : {"title" : "My updated blog post"} }

    ```
    delete 动作不能有请求体,它后面跟着的是另外一个操作,最后一个换行符不要落下。可以像 mget API 一样，在 bulk 请求的 URL 中接收默认的 /_index 或者 /_index/_type。

    ```text
    POST /index/_bulk
    { "index": { "_type": "log" }}
    { "event": "User logged in" }

    ```

    你可以覆盖元数据行中的 _index 和 _type , 但是它将使用URL中的这些元数据值作为默认值：
    ```text
    POST /index/type/_bulk
    { "index": {}}
    { "event": "User logged in" }
    { "index": { "_type": "blog" }}
    { "title": "Overriding the default type" }

    ```
    bulk API还可以在整个批量请求的最顶层使用 consistency 参数，以及在每个请求中的元数据中使用 routing 参数。
- 冲突处理

  当我们使用 index API 更新文档 ，可以一次性读取原始文档，做我们的修改，然后重新索引整个文档，如果其他人同时更改这个文档，他们的更改将丢失。Elasticsearch是分布式的，当文档创建、更新或删除时，新版本的文档必须复制到集群中的其他节点，Elasticsearch 也是异步和并发的，这意味着这些复制请求被并行发送，并且到达目的地时也许顺序是乱的。Elasticsearch 使用这_version号来确保变更以正确顺序得到执行，如果旧版本的文档在新版本之后到达，它可以被简单的忽略。

    ```text
    PUT /{index}/{type}/{id}?version=1
    {
      "field": "value",
      ...
    }
    ```
  此时，索引中的文档只有现在的 _version 为1时，本次更新才能成功。否则 Elasticsearch返回409 Conflict HTTP响应码，和一个如下所示的响应体：

    ```json
    {
      "error": {
          "root_cause": [
            {
                "type": "version_conflict_engine_exception",
                "reason": "[type][id]: version conflict, current [2], provided [1]",
                "index": "index",
                "shard": "3"
            }
          ],
          "type": "version_conflict_engine_exception",
          "reason": "[type][id]: version conflict, current [2], provided [1]",
          "index": "index",
          "shard": "3"
      },
      "status": 409
    }
    ```

  同时如果想用外部自定义的version可以url上拼接version=1&version_type=external，定义使用外部version。

