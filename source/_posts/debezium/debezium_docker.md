---
title: Debezium docker 形式部署
date: 2020-05-28 15:50:37
categories: Debezium
tags: [Debezium]
---

介绍采用 docker 形式的部署参数配置、注意事项

<!-- more -->

## 1. docker 参数介绍

- `BOOTSTRAP_SERVERS`: kafka:port,kafka2:port
  - kafka broker 地址列表
- `GROUP_ID`:
  - 分布运行模式下, 供服务间互相发现
- `CONFIG_STORAGE_TOPIC`: string
  - topic 特点
    - 单分区多副本
    - clean policy: compact (default)
  - 描述: connector、task 配置存储
    - 当配置变更时新增数据
  - 辅助参数:
    - CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 副本因子 (config.storage.replication.factor)
- `OFFSET_STORAGE_TOPIC`: string

  - topic 特点
    - 多分区多副本 (默认 25 partition, 最小 3 副本)
    - 但是`当分区数发生变化时, 会出现 bug`, connetor 配置之前完成分区配置 (bug 修复前不再更改分区数)
    - clean policy: compact
  - 描述: 每个 connector 的偏移量存储
  - 辅助参数:
    - CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 副本因子 (offset.storage.replication.factor)
    - OFFSET_FLUSH_INTERVAL_MS: Int (offset 刷新间隔)

- `STATUS_STORAGE_TOPIC`: string
  - topic 特点
    - 多分区多副本 (未测试分区增加后的情况, 是不是和上面 offset 有同样问题)
    - clean policy: compact
  - 描述: connector、task 状态存储
  - 辅助参数:
    - CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 副本因子 (status.storage.replication.factor)
- CONNECT_KEY_CONVERTER_SCHEMAS_ENABLE=false
  - converter 为 json 时关闭 schema
  - 决定表的 topic 数据, key 是否包含表结构信息
- CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE=false
  - converter 为 json 时关闭 schema
  - 决定表的 topic 数据, value 是否包含表结构信息
- CONNECT_PRODUCER_MAX_REQUEST_SIZE=20971520 (字节)
  - 作为 producer 时, 发送数据大小限制. default: 1M
  - 需要配合 kafka MESSAGE_MAX_BYTES 字段, kafka 接收数据大小限制
- CONNECT_DATABASE_HISTORY_KAFKA_RECOVERY_POLL_INTERVAL_MS=1000 （ms）
  - 恢复超时配置
  - history topic 保存 schema, connector 重启时, 需要拉取 schema 信息, 以便对数据进行特殊数据类型处理.
  - default: 100ms 忘了
- HEAP_OPTS=-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 # 需要配合 docker 内存限制使用
  - HEAP_OPTS=-Xmx512M -Xms256M
  - java 内存限制

## 3. 运行模式

- 单机模式
  - 字面意思
- 集群模式
  - 参数配置: GROUP_ID (唯一字符串, 用来 connect 服务之间互相发现)
  - 简单介绍: 当 connect 服务宕机后, 可以保证 connector 任务在其他同集群下 connect 服务中创建并运行
    - 集群多 debezium 自己完成 leader 选举

## 4. Rest api

> http://kafka.apache.org/documentation.html#connect

- GET /connectors – 返回所有正在运行的 connector 名
- POST /connectors – 新建一个 connector;请求体必须是 json 格式并且需要包含 name 字段和 config 字段，name 是 connector 的名字，config 是 json 格式，必须包含你的 connector 的配置信息。
- GET /connectors/{name} – 获取指定 connetor 的信息
- GET /connectors/{name}/config – 获取指定 connector 的配置信息
- PUT /connectors/{name}/config – 更新指定 connector 的配置信息 (完整的 config json)
- GET /connectors/{name}/status – 获取指定 connector 的状态，包括它是否在运行、停止、或者失败，如果发生错误，还会列出错误的具体信息。
- GET /connectors/{name}/tasks – 获取指定 connector 正在运行的 task。
- GET /connectors/{name}/tasks/{taskid}/status – 获取指定 connector 的 task 的状态信息
- PUT /connectors/{name}/pause – 暂停 connector 和它的 task，停止数据处理知道它被恢复。
- PUT /connectors/{name}/resume – 恢复一个被暂停的 connector
- POST /connectors/{name}/restart – 重启一个 connector，尤其是在一个 connector 运行失败的情况下比较常用
- POST /connectors/{name}/tasks/{taskId}/restart – 重启一个 task，一般是因为它运行失败才这样做。
- DELETE /connectors/{name} – 删除一个 connector，停止它的所有 task 并删除配置
- PUT /connector-plugins/{connector name: string}/config/validate
  - req: config 内容

## 5. 容器内配置文件以及 debezium 镜像定制

### 5.1 背景

通过步骤四 debezium 提供的 restful api 在创建 CDC 监控任务时，数据库信息被显示的书写，容易造成数据库信息泄露，非常危险

暴露场景如下
具体场景如下: <br>

1. 创建监控 connector (connector 是监控任务的最大单元)

```shell
Post /connectors
{
    name: '',
    config: {
        database.hostname: '',
        database.user: '',
        database.password: '',
        database.whitelist: 'dbname',
        talbe.whitelist: 'dbname.tableA,dbname.tabelB'
    }
}
```

2. 查看 connector 配置信息

```shell
GET /connectors/name/config
{
    database.hostname: '',
    database.user: '',
    database.password: '',
    database.whitelist: 'dbname',
    talbe.whitelist: 'dbname.tableA,dbname.tabelB'
}
```

3. 更新 connector 配置信息（需要全量信息，即也需要数据库信息）

```shell
Put /connectors/name/config
{
    database.hostname: '',
    database.user: '',
    database.password: '',
    database.whitelist: 'dbname',
    talbe.whitelist: 'dbname.tableA,dbname.tabelB'
}

```

> 简而言之, rest api 能够直接拿到线上的数据库配置信息

### 5.2 光放解决方案

debezium 提供了[数据库连接信息私密化](https://debezium.io/blog/2019/12/13/externalized-secrets/)的功能

#### 5.2.1 docker 容器配置

> 增加环境变量

```js
CONNECT_CONFIG_PROVIDERS = file
CONNECT_CONFIG_PROVIDERS_FILE_CLASS =
  org.apache.kafka.common.config.provider.FileConfigProvider
```

#### 5.2.2 Api 变更

使用 `${file:/kafka/dbconfig/config_name:config_key}` 形式, 来代替数据库信息, 如下所示:

``` js
Post /connectors
{
    name: '',
    config: {
        "database.hostname": "${file:/kafka/dbconfig/mysql_server_01.conf:MYSQL_SERVER_ADDRESS}",
        "database.user": "${file:/kafka/dbconfig/mysql_server_01.conf:MYSQL_SERVER_USERNAME}",
        "database.password": "${file:/kafka/dbconfig/mysql_server_01.conf:MYSQL_SERVER_PASSWORD}",
        "database.whitelist": "${file:/kafka/dbconfig/mysql_server_01.conf:MATERIAL_DATABASE_NAME}",
        "table.whitelist": "${file:/kafka/dbconfig/mysql_server_01.conf:MATERIAL_DATABASE_NAME}.shop"
    }
}
```

> `file:/kafka/dbconfig/mysql_server_01.conf` 即为配置文件在 docker 容器中的绝对路径
mysql_server_01.conf
``` yaml
MYSQL_SERVER_ADDRESS=aaa
MYSQL_SERVER_USERNAME=bbb
```
> 注意: 不要有任何其他符号, 比如引号

这样无论在 get 请求 还是 kafka 信息中, hostname\user\password, 都是如上形式进行展示 <br>
dbname 会在 kafka topic 的 offset topic 中暴露，不过没什么啥影响

### 5.3 解决方案实施

由于该方案，只支持读取容器内配置文件的方式来获取信息，所以我们的主要工作就是把数据库配置文件，放入容器中，重新制作镜像。<br>

- 方案一: 制作镜像时, 把文件放到指定位置即可
- 方案二: 更稳灵活, 在原有基础上, 添加新的 debezium 启动脚本，
  - 启动之前，去拉取配置文件服务器下载对应配置文件到置顶位置
  - 正常启动服务
  - [示例](https://github.com/houkk/debezium_docker)
