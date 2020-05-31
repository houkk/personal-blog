---
title: Mysql Connector
date: 2020-05-28 13:48:37
categories: Debezium
tags: [Debezium]
---

connector 作为 slave 连接 mysql, 接收 行级 binary log, 实现数据变更事件通知

> connector 工作流程如下

- 1. snapshot record (初始化)
  - 对数据库 加全局只读锁 做快照读, 对需要监控的数据库 schema 和 数据做同步
- 2. binlog record

<!-- more -->

### 1. Mysql 配置

- binlog_format
  - `row level`: show variables like 'binlog_format';
- version
  - `5.6 or later`: select version()
- GTID 可选
- 创建特定用户
  - 权限: GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT, LOCK TABLES ON _._ TO 'debezium' IDENTIFIED BY 'dbz';
  - select: 查找行; `仅 snapshot 需要`
  - reload: flush 命令需要; `仅 snapshot 需要`
  - show databases: 获取所有 database 名称; `仅 snapshot 需要`
  - replication slave: connect and read binlog; `总是需要`
  - replication client: SHOW MASTER STATUS, SHOW SLAVE STATUS, and SHOW BINARY LOGS; `总是需要`
  - LOCK TABLES: 表级锁; `仅 aws 需要且加锁时需要`
- mysql 不同架构
  - 单机: 没问题
  - 主从复制: connector 连接一台实例, 并且一直连接这台实例; 不同实例的 binlog 位置记录不同, 所以不可能切换实例
- mysql 数据库名，表名: ([a-z,A-Z,0-9,\_])

### 2. connector 参数

- name: 唯一名称
- connector.class: 连接器的 java class, 如果 connect 中没有内置该连接器, 需要另行下载、安装该插件
- tasks.max: 默认为 1, mysql 不能改
- database.hostname: ..
- database.port: ..
- database.user: 前面提到的`特定用户`
- database.password: ..
- database.whitelist: 逗号分隔白名单, 和 blacklist 无法共存
- database.blacklist: 逗号分隔黑名单, 和 whitelist 无法共存
- table.whitelist: .., 和 blacklist 无法共存
- table.blacklist: .., 和 whitelist 无法共存
- database.server.name:
  - 字母或下划线开头
  - mysql 集群中的唯一服务名;
  - 作为数据存储的 kafka topic 的前缀 (${server.name}.${db_name}.\${table_name})
- database.server.id:
  - mysql 集群中唯一服务 id
  - 作为 slave 同步 mysql binary log, 所以和其他 mysql slave 也要区分
- database.history.kafka.bootstrap.servers: kafka 地址 (逗号分隔)
- database.history.kafka.topic: mysql schema 变更记录 topic
  - 目前为永不删除
- database.history.store.only.monitored.tables.ddl: true or false
  - 是否监控其余表的 schema 变化并存储
  - 阿里云 rds 有本身的健康嗅探表, 会不停的生成 ddl 语句, 由于 `history topic` 不会删除, 会导致数据累积。
    - mysql.ha_health_check
    - 任务重启时, 会将 schema 拉到本地, 做数据处理.
    - 如果不停累积, 将会影响拉取以及处理的耗时, 导致其他一系列乱七八糟的问题
  - 所以, 建议设置为 true, 只监控 table.whitelist 中 schema 变更
- snapshot.mode: snapshot 类型
  - 默认(initial)会将数据库 schema 和 所有数据生成 create 事件, 传入 kafka;
  - 我们选择 schema_only(只同步 schema)
- snapshot.locking.mode
  - 是否持有全局读锁
  - innotdb 下, 全局锁和一致性读都可以用来备份, 而源码中两者重合了, 所以该选项可以设为 none
- snapshot.new.tables: parallel
  - 使得更改 table.whitlist 操作生效
  - 貌似是 `beta` 参数, 温柔修改白\黑名单, 不然会绝望的
- decimal.handling.mode: string
  - decimal/numberic 默认转为 bytes, 暂处理为 string, 更易读、处理
- gtid.new.channel.position: (new channel 即 GTID 前缀发生变化, 主要发生在 master 故障转移至其他 slave, 导致当前 slave 收到信息的 GTID 发生变化)
  - latest (default): 即当 debezium 再次连接时(若切换 master 之间暂停或断开)从最新位置开始同步数据
  - earliest: 读取`新` GTID 且未被清理的所有数据

> 示例如下

```
{
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "snapshot.locking.mode": "none",
    "database.user": "debezium",
    "database.server.id": "222333221",
    "tasks.max": "1",
    "database.history.kafka.bootstrap.servers": "192.168.4.22:9092",
    "database.history.kafka.topic": "history_inventory_1",
    "database.server.name": "mysql_inventory_1",
    "database.port": "3306",
    "table.whitelist": "inventory.orders",
    "decimal.handling.mode": "string",
    "database.hostname": "192.168.4.23",
    "database.password": "dbz",
    "name": "mysql_inventory_1",
    "database.history.store.only.monitored.tables.ddl": "true",
    "database.whitelist": "inventory",
    "snapshot.mode": "schema_only"
}
```

### 3. 启动流程

- snapshot
  - 功能:
    - 1. 设置 session 下 REPEATABLE READ 隔离级别 和全局锁 (aws 全局锁失败，针对白名单，加表级读锁)
      - `snapshot.locking.mode: none` 跳过加锁
    - 2. start transaction with consistent snapshot 快照读事务: 显示开启快照, 而不是像一般情况下, 第一次 select 开启快照读
    - 3. binglog 文件名和 position、gtid 等获取
    - 4. 实例下数据库列表获取
    - 5. 每个库的表列表获取
    - 6. connect 配置白名单数据库的 drop create 语句导入 kafka 特定 topic
    - 7. 全局锁时解锁
      - unlock tables 在针对全局，不会隐式提交
      - 针对 lock table 时，会触发隐式提交
    - 8. 对每个表进行扫描，并对每行数据生成 create 事件，导入 kafka
      - 可通过 `snapshot.mode: schema_only` 来跳过此步骤
    - 9. commit
    - 10. 再次解锁(主要针对表级锁)
  - 注意:
    - 1. 默认情况下全局锁, 当无法获得全局锁(比如 aws rds)时, 会对每个表分别加表级读锁。
    - 2. 但是表级锁的 unlock 命令会做隐式事务提交，导致 步骤 8 和 上面步骤不在同一事务中。
    - 3. 那么此时的步骤 8 就脱离了快照读的范围，此时如果有数据插入:

      - 一方面会被步骤 8 数据扫描, 创建 create 事件进入 kafka;
      - 另一方面，会在步骤 3 获取的 binlog 位置后面插入 create 记录, 然后等到 connector 快照模式完成 进入 binglog 同步模式，就会生成 Create 事件，导致重复。（或者删除操作，binlog 删除时，指向了一个不存在的 id 等）

    - 4. 所以表级锁的解锁放在事务提交后，这样如果数据库很大，就有的等了

### 4. 添加 table.whitelist

- 旧方法:
  - 1. add table
  - 2. pasue connector
  - 3. delete history topic
    - 3.1 如果此流程不起作用, 可以重来一遍，然后在此位置，手动在 topic offsets 下, 此 connector 对应的 topic 分区下，手动添加 offset 信息, 将 table.whitelist 和 table.blacklist 自定义
    - `缺点`:
      - 1. 可能会`数据重复`， 新的 offset 信息， pos 没有变化，重启后，再从 此 pos 位置读取数据，此位置后，已经存储的数据，就会出现重复。
      - 2. 如果手动插入的此条 offset pos/gtid 信息过旧，那么新表会把此 offset pos/gtid 后的所有符合表白名单的数据，同步到 kafka；`如果数据非常多`，`比较耗时`；毕竟单 connector 单 task 处理数据，每秒 3300 条所有
  - 4. update config ("snapshot.mode": "schema_only_recovery")
  - 5. restart connector\task
  - 6. write data to new table
  - 7. 恢复 snapshot.mode
- 新方法:
  - 功能仍在测试中 0.9.0 版本添加
  - connector config 中加入 "snapshot.new.tables": "parallel", (off 为关闭 whitelist 实时变更)
  - 然后通过 restful api 依次重启 connector、task
- 重点:
  - 新方法貌似不太好用, 一顿乱七八糟操作后就废了。但是把 connect 容器全部重建之后可用(重启无用), 比较诡异

> 更多信息, 比如 topic 内数据格式, 更多的数据格式兼容问题, 请查看参考文章

参考: <br>
* [debezium mysql](https://debezium.io/documentation/reference/1.1/connectors/mysql.html)
