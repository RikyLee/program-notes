
## 环境介绍
- 系统：CentOS 6.9
- MongoDB版本：[mongodb-linux-x86_64-rhel62-3.4.10](https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel62-3.4.10.tgz)

- 设备3台：172.16.10.42(27020端口)，172.16.10.90(27020端口)，172.16.10.199(27020端口)，如果没有足够设备也可部署同一台设备上面，只需要修改端口即可。

## 准备工作
- 同步系统时间：保证各个机器的时间一致，可使用/usr/sbin/ntpdate time.nist.gov 进行系统时间同步，在系统任务中添加新的任务 crontab -e
` 0 12 * * * /usr/sbin/ntpdate time.nist.gov >/dev/null 2>&1`

- 配置同时允许打开的文件最大数
  查看系统允许同时打开文件的最大数 :`ulimit -a`
  查看系统允许的最大句柄文件数:`cat /proc/sys/fs/file-max`
  修改允许最大打开文件数，修改之后会永久生效，在【/etc/security/limits.conf】中，增加下面的代码：
 ` *               soft    nofile            65536`
 ` *               hard    nofile            65536`
 
- 保证3台设备相互之间网络访问可达
- 防火墙打开27020端口

``` 
/sbin/iptables -I INPUT -p tcp --dport 27020 -j ACCEPT      #开放端口
/etc/init.d/iptables save   # 保存修改
 service iptables restart   # 重启防火墙，修改生效
```
## 准备安装
- 选择MongoDB存放位置，例如: /home/mongodb
- 使用cd 命令` cd /home/mongodb ` 进入mongodb 目录，下载MongoDB 压缩包 ` curl -O https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel62-3.4.10.tgz`
- 解压文件 `tar -xzvf mongodb-linux-x86_64-rhel62-3.4.10.tgz` 到当前目录下
- 创建数据存放文件夹 `mkdir data`,创建日志存放文件夹 `mkdir log `
- 分配机器172.16.10.42（主节点），172.16.10.90（从节点），172.16.10.199（arb仲裁节点）

## 安装MongoDB主节点
- 在mongdb 目录下新建mongod.conf文件，编辑如下内容

```
systemLog:
	destination: file
	path: /home/mongodb/log/mongod.log      #日志存放位置
	logAppend: true                                             #以追加的形式写入日志
storage:
	dbPath: /home/mongodb/data                            #数据存放地址
	journal:
		enabled: true
	directoryPerDB: true                                       #每个数据库单独一个目录
processManagement:
	fork: true
	pidFilePath: /home/mongodb/mongod.pid  #进程文件存放位置
net:
	port: 27020                                                     #mongo 占用的端口号
setParameter:
	failIndexKeyTooLong: false
  ```
- 启动mongdb服务 ` /home/mongodb/mongodb-linux-x86_64-rhel62-3.4.10/bin/mongod --config /home/mongodb/mongod.conf`

- 控制台连接mongo `/home/mongodb/mongodb-linux-x86_64-rhel62-3.4.10/bin/mongo 127.0.0.1:27020` ,必须指定端口号，因为默认的端口为27017

- 创建管理员帐号

```
use  admin;  #进入admin数据库,系统自带
db.createUser(
   {
     user: "admin",
     pwd: "admin",
     roles: [ "__system","backup","clusterAdmin","dbAdminAnyDatabase","readWriteAnyDatabase","userAdminAnyDatabase" ]
   }
);  #创建用户，并分配用户角色
```

- 查询上一步操作创建的用户 `db.system.users.find({"user":"admin"})`
查询结果如下：


- 关闭mongodb 服务  ` /home/mongodb/mongodb-linux-x86_64-rhel62-3.4.10/bin/mongod --config /home/mongodb/mongod.conf  --shutdown`

- 生成keyfile文件 `openssl rand -base64 741 > /home/mongodb/mongodb.keyfile`

- 修改主节点mongd.conf文件为如下内容

```
systemLog:
   destination: file
   path: /home/mongodb/log/mongod.log
   logAppend: true
storage:
  dbPath: /home/mongodb/data
  journal:
    enabled: true
  directoryPerDB: true
processManagement:
   fork: true
   pidFilePath: /home/mongodb/mongod.pid
net:
   port: 27020
setParameter:
  failIndexKeyTooLong: false
security:
  keyFile: /home/mongodb/mongodb.keyfile       # 使用keyfile认证
  authorization: enabled
replication:
  replSetName: mongodb_set   #名称可以自定义，但是必须保证主节点、从节点、仲裁节点统一
```

## 安装MongoDB从节点、仲裁节点
- 同理在172.16.10.90，172.16.10.199 ` /home/mongodb` 目录下新建 data、log目录

- 拷贝主节点 (172.16.10.42) `/home/mongodb` 目录下的 `mongodb.keyfile`、`mongod.conf`文件以及 `mongodb-linux-x86_64-rhel62-3.4.10` 文件夹 到从节点已经仲裁节点的`/home/mongodb` 目录下

- 修改仲裁节点` mongod.conf`` 文件内容如下

```
systemLog:
   destination: file
   path: /home/mongodb/log/mongod.log
   logAppend: true
storage:
  dbPath: /home/mongodb/data
  journal:
    enabled: false                                         # 仲裁节点本地不保存数据
  directoryPerDB: true
processManagement:
   fork: true
   pidFilePath: /home/mongodb/mongod.pid
net:
   port: 27020
setParameter:
  failIndexKeyTooLong: false
security:
  keyFile: /home/mongodb/mongodb.keyfile       # 使用keyfile认证
  authorization: enabled
replication:
  replSetName: mongodb_set   #名称可以自定义，但是必须保证主节点、从节点、仲裁节点统一
```

## 配置主节点、从节点、仲裁节点

- - 分别在三台设备上执行 ` /home/mongodb/mongodb-linux-x86_64-rhel62-3.4.10/bin/mongod --config /home/mongodb/mongod.conf` 启动mongodb服务

- 控制台连接主节点(172.16.10.42)mongo `/home/mongodb/mongodb-linux-x86_64-rhel62-3.4.10/bin/mongo 127.0.0.1:27020 -u admin -p ` , 使用admin帐号密码登录mongodb
- 初始化副本集配置

```
use admin;
config={_id:"mongodb_set",members:[{_id:0,host:"172.16.10.42","priority":20}]}
rs.initiate(config); 
```
确认返回的是{ "ok" : 1 }
上面config里面，是当前主节点对外的ip，即从节点以及仲裁节点能够访问到的ip
查看集群节点的状态:`rs.status();`

- 添加仲裁节点

`rs.addArb("172.16.10.199:27020");`

- 添加从节点

`rs.add("172.16.10.90:27020");`

- 查看集群配置`rs.config();`
显示结果如下

```
mongodb_set:PRIMARY> rs.config()
{
        "_id" : "mongodb_set",
        "version" : 4,
        "protocolVersion" : NumberLong(1),
        "members" : [
                {
                        "_id" : 0,
                        "host" : "172.16.10.42:27020",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 20,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                },
                {
                        "_id" : 1,
                        "host" : "172.16.10.199:27020",
                        "arbiterOnly" : true,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                },
                {
                        "_id" : 2,
                        "host" : "172.16.10.90:27020",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                }
        ],
        "settings" : {
                "chainingAllowed" : true,
                "heartbeatIntervalMillis" : 2000,
                "heartbeatTimeoutSecs" : 10,
                "electionTimeoutMillis" : 10000,
                "catchUpTimeoutMillis" : 60000,
                "getLastErrorModes" : {

                },
                "getLastErrorDefaults" : {
                        "w" : 1,
                        "wtimeout" : 0
                },
                "replicaSetId" : ObjectId("59f82d3d21b782e865dc270a")
        }
}
```
- 修改从节点为只读节点
获取当前配置，修改之后重新写入配置 

```
cfg = rs.conf();
cfg.members[2].priority=0
rs.reconfig(cfg);
```
-  修改从节点为永远不可能被选为主节点（非必须）
获取当前配置，修改之后重新写入配置 

```
cfg = rs.conf();
cfg.members[2].votes=0
rs.reconfig(cfg);
```

所有配置到此结束
