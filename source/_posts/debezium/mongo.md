---
title: Mongo Connector
date: 2020-05-28 15:48:37
categories: Debezium
tags: [Debezium]
---

依赖于 mongodb 的复制(副本集), 重放 oplog 实现数据变更行捕捉

<!-- more -->

## 1. connector

- 1. 依赖于 mongodb 的副本集, 利用 oplog 完成数据同步
- 2. 读取副本集的 primary node, 根据副本集状态, 始终保持连接 primary node (所以确保副本集的注册 host 可以被外部访问)
  - 副本集模式下所有的副本地址(rs.add 的地址)必须保证 connector 可以访问
  - 分片模式:
    - 要保证 mongs host、mongos --configdb 配置的 config host、mongos 添加的分片 host (sh.addShard)、 config server 副本集 (rs.add 的 host)、shard 副本集（rs.add 的 host）全部可以被 connector 访问
    - 要保证 config server、分片下拥有特殊权限用户
      - 另外, 分片下用户无法通过 mongos 创建, 需要分别创建
- 3. 一个 collection 一个 topic, 和副本集、分片数无关
- 4. topic 分区方面
  - 可以确保同一 key 进入同一 topic, 即可以确保每个 document 操作的顺序性, 所以分区没什么影响
- 5. 配置项
  - 0. mongodb.hosts (host:port)
    - 副本集模式: 主副本 host, 其实无所谓 (副本集的主副并不固定, 内部自行选举)
    - 分片副本集: config server host
  - 1. 指数规避配置 (暂时可以不予关注)
    - connect.backoff.initial.delay.ms: 1s
    - connect.backoff.max.delay.ms: 120s
    - connect.max.attempts: 16
  - 2. snapshot.mode
    - connector 连接时是否进行数据备份
    - initial: 进行过往数据 copy
    - never: 只记录新数据
  - 3. tasks.max
    - 默认 1
    - 大于等于分片数
  - 4. tombstones.on.delete
    - 墓碑事件
  - 5. mongodb.user
  - 6. mongodb.password
  - 7. mongodb.name:
    - 类似 mysql/pg connector 的 database.server.name, 唯一、kafka topic 前缀
  - 8. database.whitelist
  - 9. collection.whitelist
- 注意事项:
  - 命名规则
    - 1. 服务名 /[a-z,A-Z,\_][a-z,a-z,0-9,\_]+/
    -

> config demo

```
{
    "connector.class": "io.debezium.connector.mongodb.MongoDbConnector",
	  "mongodb.hosts": "192.168.4.23:27017",
	  "mongodb.name": "mongo_01",
    "mongodb.user": "debezium",
    "mongodb.password": "dbz",
    "database.whitelist": "inventory",
    "collection.whitelist": "inventory.customers,inventory.orders",
    "snapshot.mode": "never",
    "tasks.max": "1"
}
```

## 2. database

- 集群模式
  - 分片模式和副本集模式(需要 oplog)
  - 需要有权限读取 oplog 集合的账户
- 注意事项:
  - 命名规则
  - 副本集模式
    - 内部权限 (keyfile)
    - 保证 rs.add 的每个节点地址, connector 都可以访问

> 特殊权限用户, 每个分片都需要(感觉这一点有点问题，太繁琐了，是用错了，还是怎么回事)

```
    use admin;

    db.runCommand({
        createRole: "listDatabases",
        privileges: [
            { resource: { cluster : true }, actions: ["listDatabases"]}
        ],
        roles: []
    });

    db.createUser({
        user: 'debezium',
        pwd: 'dbz',
        roles: [
            { role: "readWrite", db: "inventory" }, // inventory 为监控库
            { role: "read", db: "local" },
            { role: "listDatabases", db: "admin" },
            { role: "read", db: "config" },
            { role: "read", db: "admin" }
        ]
    });
```

## 3. 流程

- Initial sync
  - 初始化同步 oplog
- Tailing the oplog
  - 拥有 offset 后, 读取 oplog

## 4. topic payload 构成

- after: init 和 create 时的完整数据
- patch: update 时的 sql
- op:
  - r: snaphsot init
  - u: update
  - d: delete
  - c: create

### 1. init snapshot

> operation: r

```
"payload": {
    "after": "{\"_id\": {\"$numberLong\": \"1001\"},\"first_name\": \"Sally12\",\"last_name\": \"Thomas\",\"email\": \"sally.thomas@acme.com\"}",
    "patch": null,
    "source": {
      "version": "0.10.0.Final",
      "connector": "mongodb",
      "name": "mongo_02",
      "ts_ms": 1574235688000,
      "snapshot": "true",
      "db": "inventory",
      "rs": "rs0",
      "collection": "customers",
      "ord": 1,
      "h": 5175105543699020230
    },
    "op": "r",
    "ts_ms": 1574235691071
  }
```

### 2. create

> operation: c

```
"payload": {
    "after": "{\"_id\": {\"$oid\": \"5dd4ef7e9e9939487b6c0239\"},\"first_name\": 3.0,\"last_name\": 4.0,\"email\": 5.0}",
    "patch": null,
    "source": {
      "version": "0.10.0.Final",
      "connector": "mongodb",
      "name": "mongo_02",
      "ts_ms": 1574236030000,
      "snapshot": "false",
      "db": "inventory",
      "rs": "rs0",
      "collection": "customers",
      "ord": 1,
      "h": 131523180395089317
    },
    "op": "c",
    "ts_ms": 1574236030371
  }
```

### 3. update

> operation: u

> patch: 显示 set 操作, 为了保证 oplog 操作的幂等性, 保证重复执行后的正确性

> 注意: patch 内容由 mongodb 提供, 且 mongodb 各个版本之间的 patch 可能存在不同

```
  "payload": {
    "after": null,
    "patch": "{\"$v\": 1,\"$set\": {\"last_name\": 5.0}}",
    "source": {
      "version": "0.10.0.Final",
      "connector": "mongodb",
      "name": "mongo_02",
      "ts_ms": 1574236075000,
      "snapshot": "false",
      "db": "inventory",
      "rs": "rs0",
      "collection": "customers",
      "ord": 1,
      "h": 5737963839219456046
    },
    "op": "u",
    "ts_ms": 1574236075380
  }
```

### 4. delete

> operation: d

```
  "payload": {
    "after": null,
    "patch": null,
    "source": {
      "version": "0.10.0.Final",
      "connector": "mongodb",
      "name": "mongo_02",
      "ts_ms": 1574237481000,
      "snapshot": "false",
      "db": "inventory",
      "rs": "rs0",
      "collection": "customers",
      "ord": 1,
      "h": -3225175849180220485
    },
    "op": "d",
    "ts_ms": 1574237481422
  }
```
