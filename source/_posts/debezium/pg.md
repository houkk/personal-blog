---
title: PostgreSql Connector
date: 2020-05-28 14:48:37
categories: Debezium
tags: [Debezium]
---

基于 pg 的逻辑复制功能, 需要 wal(logic) 和 逻辑解码输出插件的辅助, 实现数据变更事件通知

<!-- more -->

## 1. PostgreSql 配置

- 1. 版本: 9.6 or later (10.0 以上最好)
- 2. wal (write-ahead logs 预写日志)
  - `xlog`: pg 9.x 的 wal 叫做 xlog
  - lsn(localtion 9.x): lsn 日志号, 标记 wal 文件以及文件内偏移量
    - 32位/32位
    - 左边 32 位，通过计算获得 wal 日志的文件名: "timeline + lsn计算"
    - 右边 32 位，获得偏移量
  - 逻辑解码
  - output plugins
  - replication slots
  - wal_level: logic
    - wal 日志中加入信息支持逻辑解码, 增大 wal 日志, 尤其是 update、delete
    - 不支持 DDL、列存、压缩表的解码, 需要额外触发器处理 DDL, 可以做到表级订阅
  - 不使用的 replication slots 要及时删除
    - replication slots 存在即保证从库宕机后, 主库不会删除从库上报的 lsn 之后的 wal
    - connector 的 上报 lsn 不变化, 造成 xlog 堆积, 影响磁盘空间
- 3. 逻辑解码输出插件
  - 解码并输出
  - pgoutput, 10+ 预装了
  - wal2json, 基于 json, 由 wal2json 社区维护, 需要预装
  - decoderbufs, 基于 protobuf, 由 debezium 社区维护，需要预装
- 4. 数据库设置
  - 1. wal_level: logical
  - 2. max_replication_slots \>= 1
  - 3. 权限
    - REPLICATION
    - LOGIN
  - 4. postgresql.conf
    - shared_preload_libraries = 'decoderbufs,wal2json' (预装插件配置)
    - wal_level = logical
    - max_wal_senders = 1
    - max_replication_slots = 1
- 5. 表设置
  - 1. REPLICA IDENTITY (副本身份, 决定了在 kafka message 中 update 和 delete 事件 的 before 内容)
    - DEFAULT: 仅显示主键
    - NOTHING: nothing
    - FULL: 全部信息
    - INDEX: 索引信息
    - 查看 sql:
- 6. 数据库字符集
  - 仅支持 UTF-8 字符集
    - connector 无法处理 单字节字符集中的 包含扩展 ASCII 码字符集的字符串 (`TODO`: 不太理解, 后续跟进)
    - Debezium currently supports only database with UTF-8 character encoding. With a single byte character encoding it is not possible to correctly process strings containing extended ASCII code characters.

- 7. 常用命令合集

``` sql
// superuser 创建 publication, 使用 pgoutput 时需要事先创建
CREATE PUBLICATION ${name} FOR ALL TABLES

// publication
select * from pg_publication;

// show publication tables
select * from pg_publication_tables;

// 复制插槽 列表
SELECT * FROM pg_replication_slots;

// wal 等级
show wal_level;

// 新安装插件
SHOW shared_preload_libraries;

// 查看 REPLICA IDENTITY
select
	case relreplident
		when 'd' then 'default'
		when 'n' then 'nothing'
		when 'f' then 'full'
		when 'i' then 'index'
	end as replica_indentity,
	relname
from pg_class where relname in ('products', 'geom', 'orders', 'customers');

// 修改 REPLICA IDENTITY
alter table test_debezium_1 replica IDENTITY full
```

### 1.1 各平台配置流程

- heroku 无法安装插件, 只能使用 pg 10+ 自带的逻辑解码
- Amazon RDS
  - 1. rds.logical_replication: 1 (`TODO:` 此参数的修改意义, 貌似是开关)
  - 2. wal_level: logical; 此参数无法修改, 步骤一修改后自动生效(或需重启)
  - 3. plugin.name 修改
    - 10+ 默认 pgoutput
  - 4. 版本限制: wal2json 版本太低, 类型约束信息不完整
    - Postgres 9.6: 9.6.10 and newer
    - Postgres 10: 10.5 and newer
    - Postgres 11: any version

## 2. connector

### 2.1 逻辑解码插件

> 需要安装在 postgresql server 中, 用作解码 wal 为可读 (创建 replication slot 时亦会使用）

> [细节](https://debezium.io/documentation/reference/0.10/postgres-plugins.html)

- [decoderbufs](https://github.com/debezium/postgres-decoderbufs)
  - Debezium 社区维护, 基于 ProtoBuf
- [wal2json](https://github.com/eulerto/wal2json)
  - wal2json 社区维护, 基于 json
  - [一堆问题, 乱七八糟的](https://debezium.io/documentation/reference/0.10/connectors/postgresql.html#discrepance-between-plugins)
- pgoutput
  - postgresql 10+ 自带 (Debezium 0.10 可用)
  - 满足条件的话，直接使用这个; (其余是 pg 10 之前的版本使用)

### 2.2 Connector

> 注意事项

- connector 依赖于 pg 的逻辑解码功能, 所以:
  - `无法监控 DDL 变化`
  - 逻辑解码器原因, connector 只能监控集群中`主服务器`变化
    - 主服务宕机或被降级, connector 停止
    - 如果此服务器会被恢复, 重启 connector 即可
    - 如果其他服务器升级为主服务器， connector 重启前需要变更配置
- 主键更改有风险: [流式修改](https://debezium.io/documentation/reference/0.10/connectors/postgresql.html#streaming-changes) -> Note (TODO: 搞清楚怎么回事)
  - 貌似没什么影响
    - 无主键的表, 数据没有 key, 不是为 null
    - 将主键去除后(不是删列), 数据 key 仍为原来的 key 结构
- 当 replication slot 丢失时, 需要重启 task 来自动 initReplicationSlot

### 2.3 connector 工作流程

- snapshot
  - 1. 设置事务隔离级别 SERIALIZABLE, READ ONLY, DEFERRABLE (DEFERRABLE 保证了长时间执行时不与其他串行事务冲突)
  - 2. 对每张表加锁 SHARE UPDATE EXCLUSIVE MODE (锁有点乱, TODO: 整理并详解), 确保表结构不被更改
  - 3. 获取当前事务开始时 wal(lsn) 位置, 为了在 snaphsot 后从该位置开始监控数据, 防止数据遗漏
  - 4. 获取 schema, 如果开启数据同步, 同时会为每行现有数据生成 read 事件, 同步至 kafka 对应 topic
  - 5. commit
  - 6. 记录快照完成 offset (? 和 步骤 3 的关系 ? 为了 snapshot 未完成时意外重启重新开始)
- Streaming Changes
  - initReplicationSlot (replication slot, 没有则创建, 判断是否被其他进程占用)
  - initPublication (pgoutput 插件需要 `publication` 来进行限定表白名单, 没有则创建 all tables)
    - 可以事先用 superuser 创建, connector 默认 publication 为 `dbz_publication`
  - 从上述步骤 3 中获取到的 LSNs (Log Sequence Numbers) 开始

### 2.4 类型转换

> 大体类型转换和 mysql 类似, 下面列出特殊点

- HSTORE => String ('"a" => "b"' => "{"a": "b"}")
- 网络地址类型
  - INET、CIDR、MACADDR、MACADDR8 => String
- PostGIS Types (空间类型转化) (待续... 有空再说)
- Toasted: (存储内容超过 8 kb, 采用 Toasted 存储, 具体再说吧)

### 2.5 部署

- 1. 安装逻辑解码插件 (decoderbufs、wal2json ...)
  - pg 10+ 默认安装 pgoutput, 推荐使用 (pgoutput 需要 superuser 权限创建 publication, 或者事先创建)
- 2. 配置 pg 支持逻辑复制 (见第一大项)
- 3. POST 创建 connector
  - `name`: '' (connector 名称, 唯一性)
  - `config`:
    - `connector.class`: "io.debezium.connector.postgresql.PostgresConnector"
    - `tasks.max`: 1
    - `plugin.name`: 见步骤一
    - `slot.name`: 逻辑解码槽, 保证 wal 不被清理, 所以, 要特别注意 (和 connector 一对一)
    - `publication`: 存在默认值, dbz_publication;
      - 使用 pgoutput 时, 需创建;
      - 需要使用 supuser 创建, 由于目前功能缺陷, 可以事先创建 all tables 的 publication
    - `slot.drop_on_stop`: true or false
      - connector 停止后 (stop 实例 ?), 是否删除 replication slot
      - 推荐测试环境设置为 true, 防止 wal 大量堆积, 占用磁盘空间
      - 生产环境设置为 false, 但一定要有 connector 宕机的处理预案以及磁盘空间预估, 不然准备迎接磁盘`爆满`吧
      - **_`注意`_** : 当为 true 时, connector 重启会删除、重建 slot, 会导致`数据丢失`
    - `database.hostname`
    - `database.port`
    - `database.user`
    - `database.password`
    - `database.dbname`: 库名
    - `database.server.name`: 服务名, 唯一性
      - topic name: `${database.server.name}.${schema name}.${table name}`
    - `schema.whitelis`t / `blacklist`
    - `table.whitelist` / `blacklist`
    - `column.blacklist`: schemaName.tableName.columnName
    - `decimal.handling.mode`: "string" (Decimal 以 string 形式存在)
    - `tombstones.on.delete`: true or false (kafka 墓碑事件)
    - `snapshot.mode`:
      - `initial`(default): 仅在 connector 创建时执行快照
      - `always`: connector `每次启动`时执行快照
      - `never`: 字面意思
      - `initial_only`: 只执行快照, 后续的数据变化流不做处理
      - `exported`: 从 replication slot 创建的时间节点开始执行, 无锁
      - `custom`: 自己写代码, 同时需要制定 `snapshot.custom.class`
    - `snapshot.lock.timeout.ms`: 10000 (获取表锁超时时间)

```
{
    "name": "pg_inventory_1",
    "config": {
	    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
	    "database.user": "postgres",
	    "database.dbname": "postgres",
	    "slot.name": "pg_inventory_1",
	    "database.server.name": "pg_inventory_1",
	    "database.port": "5432",
	    "plugin.name": "pgoutput",
	    "schema.whitelist": "inventory",
	    "table.whitelist": "inventory.orders",
	    "slot.drop_on_stop": "true",
	    "decimal.handling.mode": "string",
	    "database.hostname": "192.168.4.21",
	    "database.password": "postgres",
	    "snapshot.mode": "never"
	}
}
```

## 3. 问题

- 1. 更新配置时, 报错, 且出现数据丢失
  - 复现步骤
    - 一切正常后
    - 没 200ms, update\delete\insert pg
    - kafka 数据正常
    - 更新 table.whitelist
    - 待更新完成后, 发现数据丢失
  - 原因: slot.drop_on_stop: true 时, connector 重启, 则 slot 删除后重建, 造成数据丢失

参考文档: <br>

1. [逻辑复制](https://www.postgresql.org/docs/10/logical-replication.html) <br>
2. [pg 10.0 逻辑复制](https://juejin.im/entry/58ba3279570c350062124bc9) <br>
3. [PG JDBC logical replication API](https://jdbc.postgresql.org/documentation/head/replication.html) <br>
