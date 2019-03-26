---
title: NoSQL 数据库的种类
category: 数据库
date: 2019-03-26 14:00
description: 简述非关系性数据库
---

在性能，可扩展性，灵活性等方面，NoSQL 要明显强于传统的关系型数据库。
但不遵循关系数据模型也是 NoSQL 的劣势。

NoSQL 主要分为以下四类

- key-value store
- colume store
- Document Database
- Graph Database

# Key-Value Store

最简单的存储类型，每个 Item 以 Key-Value pair 的形式存储在数据库中。
比较典型的有 Riak, Voldemort 和 Redis.

# Column Store

数据的列的形式存储而不是以行的形式，可以做到在超大数据集上的查询优化。
典型的有 Cassandra 和 HBase

# Document Database

每个 key 与一个复杂的数据结构成对保存。该数据结构成为 Document(文档), Document 可以保存许多不同的 k-v pair, key-array 对，或者签到 Document 结构。
典型的有 MongoDB

# Graph Database

用于网络信息，比如社会关系。
典型的有 Neo4J, HyperGraphDB.
