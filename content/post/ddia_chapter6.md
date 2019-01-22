---
title: "Designing Data-Intensive Applications 6: Partition"
date: 2019-01-21T16:49:01+08:00
draft: false
---

# Partition

- **Partition 两种主流方式**

  1. partition by key range: efficient at range query but may got skewed and hotspot issue
  2. partition by hashing key: less skewed and hostspot issue but sacrifices range query performance

- **Partitioning secondary indexes**

  1. By Document(local index): Cassandra， MongoDB， ElasticSearch，VoltDB，Riak
  2. By Term(global index): DynamoDB
  3. pros and cons of the two methods

- **Rebalaning partitions**

  1. 解释下为啥需要 rebalancing
  2. rebalancing 的目标

     - When rebalancing, the database should continue accept read writes requests.
     - No more data than necessay should be moved during reblancing.
     - After rebalancing, the load (data storage and read, write requests) should be shared fairly amond nodes.

  3. rebalancing 的三种 partition 模式：

     - fixed partitioning: Riak, ElastichSearch, CouchBase, Voldermot
     - dynamic partition: HBase and MongoDB
     - Partitioning proportionally to Nodes: Cassandra
     - 三者对比，优缺点，适用场景

  4. Operations, manual or automatic: 了解下 fully automatic 存在的缺陷 - cascading faliure

- **Request Routing**

  1. 三种模式

     - Allow clients to send requests to all nodes, using round-robin
     - Create a routing tier
     - Require client to be aware of partitioning and how the partitioning is conducted

  2. 三种方式都有一个难点：how does the component making the routing decision (which may be one of the nodes, or the routing tier, or the client) learn about changes in the assignment of partitions to nodes

     1. MongoDB, HBase 等用方式 2，依赖 zookeeper 做管理
     2. Cassandra 用了自己的 goosip protocol

- **MPP (Massively parallel processing) 了解下**
