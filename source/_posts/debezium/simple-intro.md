---
title: Debezium 介绍
date: 2020-05-28 10:48:37
categories: Debezium
tags: [Debezium]
---

* 分布式 CDC 服务
* 生成数据库行级变化事件流的分布式平台
* 基于 kafka connect
* 位于 kafka 和 其他数据系统之间的数据工具, 内置 producer 和 consumer 接口, 提供插件模式(供使用者自定义开发 connector、converter等, 将数据导入导出 kafka、格式化等)

> 项目很好, 问题修复非常及时
> 大体构成: debezium( connector( task))

<!-- more -->

> kafka connect 的 CDC 架构如下

![基于 kafka connect 的 CDC 流转](https://personal-1258258052.cos.ap-shanghai.myqcloud.com/debezium/debezium_arch.png)

> debezium 就是对 kafka connect 进行包装, 并实现了上图读取数据库变更部分
> 所以, 建议先了解 [kafka connect](https://docs.confluent.io/current/connect/index.html), 默认已有所了解

## 1. 构成

* debezium 集群
  * 没有仔细研究, 内置 leader 选举算法, 不需要过多关注
  * connector
    * 功能上是 CDC 的功能实现 (debezium 推出了多种数据库的 connector 实现)
    * 架构上是 CDC 任务的管理者
  * task
    * CDC 的执行者
* kafka 集群
  * 存储 debezium 的一系列信息
    * config topic: connctor 配置信息
    * status topic: connector、task 的运行状态
    * offset topic: cdc 对于数据库的 offset 信息 (比如 mysql binlog 的 pos gtid 等)
  * CDC 数据
    * 内置 kafka 的 sink, 直接落地 kafka, 为后续的操作提供方便
    * 表和 topic 一对一

## 2. 功能特点

* 保证捕获所有变更
  * 以 binlog row 模式为例, 只要 binlog 不丢, 就可以保证捕获所有变更
* 低延迟
* Snapshots
  * 已有数据的 snapshot
  * 个人建议不要理会这个, 感觉这功能放在 CDC 中不太合适
* Filters
  * 自定义 db table column 的过滤
* 服务性能监控
  * JMX, 这个没什么好说的
* 消息转换
  * 没了解过啊, 有兴趣的同学可以试试

## 3. 主要原理

> 简单的介绍一下 debezium 的 CDC 是怎么做的:
基于数据库本身的逻辑记录和备份机制来实现的

举几个例子

* Mysql
  * binlog(row)
* pg
  * wal(logic)
  * Replication Slot
* mongodb
  * oplog

> 所以, 就是逻辑重放

## 4. 数据库支持

> 提供了多种数据库的支持

* Mysql
* MongoDB
* PostgreSql
* Oracle
* Sql Server
* Db2
* Cassandra

## 5. Demo

[Debezium 官网](https://debezium.io/documentation/reference/1.1/tutorial.html)
