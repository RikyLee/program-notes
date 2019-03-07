# RocketMQ 安装配置

## RocketMQ 安装

- 安装Java8+
- [官网](http://rocketmq.apache.org/docs/quick-start/)下载最新版本的RocketMQ二进制包rocketmq-all-4.3.2-bin-release.zip

- 解压到用户根目录并进入rocketmq-all-4.3.0-bin-release目录 `unzip rocketmq-all-4.3.2-bin-release.zip &&　cd rocketmq-all-4.3.0-bin-release`

- 单节点启动RocketMQ服务
  - 后台启动NameServer服务 `nohup sh bin/mqnamesrv &`
  - 后台启动Broker服务，使用-n参数指定NameServer服务器的地址,-c 指定配置文件位置 `nohup sh bin/mqbroker -n localhost:9876 &`

- 停止RocketMQ服务
  - 停止Broker服务 `sh bin/mqshutdown broker`
  - 停止NameServer服务 `sh bin/mqshutdown namesrv`

- 开放防火墙端口允许访问9876端口

## RocketMQ配置文件详解

- brokerClusterName 所属集群的名字,如果有多个master，那么每个master配置的名字应该一样 `brokerClusterName=default`
- brokerName 当前broker的名称，集群内每个broker的名称都必须不一样 `brokerName=broker-a`
- brokerId broker运行方式0 表示Master, >0 表示slave `brokerId=0`
- namesrvAddr nameServer 地址，分号分割，broker启动时会跟nameserver建一个长连接，broker通过长连接才会向nameserver发新建的topic主题，然后java的客户端才能跟nameserver端发起长连接，向nameserver索取topic，找到topic主题之后，判断其所属的broker，建立长连接进行通讯 `namesrvAddr=10.3.175.214:9876;10.3.175.215:9876`
- defaultTopicQueueNums 在发送消息时，自动创建服务器不存在的Topic，默认创建的队列数 `defaultTopicQueueNums=4`
- autoCreateTopicEnable 是否允许Broker 自动创建Topic，建议线下开启，线上关闭`autoCreateTopicEnable=true`

- autoCreateSubscriptionGroup 是否允许Broker自动创建订阅组，建议线下开启，线上关闭`autoCreateSubscriptionGroup=true`

- listenPort Broker 对外服务的监听端口 `listenPort=10911`

- deleteWhen 删除文件时间点，默认是凌晨4点 `deleteWhen=04`

- fileReservedTime 文件保留时间，默认48小时 `fileReservedTime=120`

- mapedFileSizeCommitLog commitLog每个文件的大小默认1G,消息实际存储位置，和ConsumeQueue是mq的核心存储概念，用于数据存储，consumequeue是一个逻辑的概念，消息过来之后，consumequeue并不是把消息所有保存起来，而是记录一个数据的位置，记录好之后再把消息存到commitlog文件里 `mapedFileSizeCommitLog=1073741824`

- mapedFileSizeConsumeQueue ConsumeQueue每个文件默认存30W条，根据业务情况调整`mapedFileSizeConsumeQueue=300000`

- destroyMapedFileIntervalForcibly 表示第一次拒绝删除之后能保留的最大时间,在清除过期文件时，如果该文件被其他线程所占用（引用次数大于0，比如读取消息），此时会阻止此次删除任务，同时在第一次试图删除该文件时记录当前时间戳,超过该时间间隔后，文件将被强制删除,默认

- diskMaxUsedSpaceRatio 设置磁盘最大使用率上限阀值百分比，表示commitlog、consumequeue文件所在磁盘分区的最大使用量,如果超过该值，则需要立即清除过期文件 `diskMaxUsedSpaceRatio=88`

- diskSpaceWarningLevelRatio 如果磁盘分区使用率超过该阔值，将设置磁盘不可写，此时会拒绝新消息的写入 `diskSpaceWarningLevelRatio=0.90`
- diskSpaceCleanForciblyRatio 如果磁盘分区使用超过该阔值，建议立即执行过期文件清除，但不会拒绝新消息的写入 `diskSpaceCleanForciblyRatio=0.85`

- storePathRootDir 存储路径 `storePathRootDir=/usr/local/rocketmq/store`

- storePathCommitLog commitLog存储路径 `storePathCommitLog=/usr/local/rocketmq/store/commitlog`

- storePathConsumeQueue  消费队列存储路径 `storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue`
- storePathIndex 消息索引存储路径 `storePathIndex=/usr/local/rocketmq/store/index`

- storeCheckpoint checkpoint 文件存储路径 `storeCheckpoint=/usr/local/rocketmq/store/checkpoint`

- abortFile abort 文件存储路径 `abortFile=/usr/local/rocketmq/store/abort`

- maxMessageSize 限制的消息大小 `maxMessageSize=65536`

- brokerRole  broker的角色:ASYNC_MASTER 异步复制Master,SYNC_MASTER 同步双写Master,SLAVE
- flushDiskType 刷盘方式:ASYNC_FLUSH 异步刷盘,SYNC_FLUSH 同步刷盘
- sendMessageThreadPoolNums 发送消息的线程池数量
- useReentrantLockWhenPutMessage 发送消息是否使用可重入锁
- waitTimeMillsInSendQueue  发送消息入队等待时间
- pullMessageTreadPoolNums 拉消息线程池数量
- checkTransactionMessageEnable 检查是否事物消息
- brokerIP1 强制指定本机IP，需要根据每台机器进行修改。官方介绍可为空，系统默认自动识别，但多网卡时IP地址可能读取错误