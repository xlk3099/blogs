---
title: "Designing Data-Intensive Applications 5: Replication"
date: 2019-01-19T17:48:02+08:00
draft: false
categories: ["tech", "读书"]
---

# Replication

## 定义

Keep a copy of the same data on multiple machines via networks

## 作用

**Increase availability**

- keep the system running, even one machine (or several machines, or even one data center) goes down
- place data geographically close to users, so users can interact with the data faster

**Increase reliability**

- allow disconnected operation: allow an application continue working even when there is a network interruption. e.g. 日历本

**Increase scalability**

- being able to handle a higher volume of reads than a single node, by performing loads on replicas

## 三种模式

1. **Single-Leader replication**: clients send all data to a single node, which sends a stream of data changes to its followers. Reads can be performed on any replicas, but reads can be **stale**
2. **Multi-reader replication**: clients send each write to one of several leaders nodes, **any** of which can accept writes, the leaders send streams of data changes events to any other follower nodes
3. **Leaderless replication**: clients send each writes to several nodes, and read from **several nodes in parallel** in order to detect and correct nodes with stale daa.

---

## Leader based replication 问题集合

- **概括 Synchronous vs asynchronous replication 的优劣?**

      通常情况下 replication 速度是非常快的，但某些情况会慢（比如网络故障），
      Sync replication：保证所有的 follower 都跟 leader 数据一致，缺点一旦任意一个节点 fail，那么 write operation 就不能被处理，还有一个是 performance 慢，因为要等全部节点都更新完。
      Async replication：优势快，能 handle 部分 follower 挂掉的情况，缺点：replication lag 明显的时候，会存在不同 follower inconsistency。

      整体而言，现在普遍采用的是 async replication 方式，追求 eventual consistency。

- **setup new follower 的时候，如何保证 follower 数据跟 leader 一致?**
  如果是简单的复制 leader 的数据，那么在 copy 的同时，leader 的数据如果发生变动，比如接受新的更新需求（update， delete， write），那么就不能保证跟 leader 的一致。简单粗暴的方法，锁住 leader 的数据库，但这违背了 **high availability**的原理
  一般步骤

      1. 给 leader 拍摄快照
      2. copy 快照到 follower
      3. follower 向 leader 要求快照后的数据
      4. 当 follower 处理完从快照完之后的 backlog of data changes，就说 follower 已经 catch up

- **如何处理 node outages？ （3 个点）**

      如果是 follower，因为每个 follower 都有自己的 log，所以恢复比较容易，直接继续从 leader 那边 pull 即可

      如果是 leader，需要进行 failover 的操作：

      1. 确定 leader fail： 参考 timeout 时间，timeout 过了都没收到 leader 的 response，确认 leader fail
      2. 从 follower 里面选取一个节点作为 leader，当然这个选取也是有讲究的。
      3. Recofing the system 采用新的 leader
         - client now sends data to the new leader
         - ensure the older leader now becomes follower

- **解释下 leader failover 非常容易出错的原因（4 个点）**

      1. 主流基本都是 async replication，代表着 new leader 有可能还没收到 old leader 最新的全部数据，这时候 new leader 可能会面临 conflicts writing。常用方式，discard old leader unreplicated writes，但这样对 client 的 durability 造成影响，client 以为发送了，结果没入库。。。

      2. 此外，discard 部分数据也有影响，尤其当有其它数据库系统连接 old leader 的时候，比如在其它数据系统已经收入了某些 key，如果 discard 了那些内容，重新写，或者接收到数据更新时，会出现数据不一致，尤其是那些 increment 操作。

      3. 两个 nodes 相信他们都是 leader 的处理方式。

      4. 如何设计合理的 timeout 也是需要考量，短了吧，有很多不必要的麻烦，长了吧，系统的 availability 就受到影响。

         整体来说，leader 的 failover 没有太好的处理方式，所以很多组偏向人工解决

- **解释下 replication logs 的几种实现方式 以及优劣 （3 种）**

      1.  **Statment based replication log**

          leader logs every request it executes and sends the statement log to its followers.

         缺点：

         - 有些 non-determinstic statement 在不同的节点会导致不同的结果，比如 random()，now()
         - 又有些 auto increment column 依赖已经存在的数据的话，这样也要求 followers 严格要求跟 leader 一致的顺序，这在 follower 并发处理中受到了限制。

         MySQL 5.1 之前是采取的这种模式，之后取消，使用了 row based

      2.  **Write Ahead Log shipping （WAL）** 模式

         Leader sends WAL 到它的 follower 去，这种模式在 oracle 数据库里面被广泛使用

         缺点：如果 follower 的 db engine 换了，或者说版本更新等，那就无法使用。

      3. **Logical（row-based） replication** **（加星，使用最广）**

         logical log format 跟 WAL 的 log format 格式不一致，

         好处：支持 leader-follower 不同版本的更新，使用不同的 engine，支持 zero-downtime upgrade

         而且三方 tools 更好 parse。

         缺点：more prone to bugs

- **解释有 replication lag 高的情况下，要追求三种状态 （3 种）**

      replication lag 主要是导致不同 nodes 的状态不一致。

      1. **Read after write consitency**： Users should always see data that they submitted themselves.

         为啥会产生在 write 之后，读取到的数据跟刚 write 数据不一致，比如写的是 leader db，读的是其中的一个 follower db。

         - 对于用户本身，那可以让他继续从 leader 读
         - 用户写完开始读取的时候，track last update 时间，如果小于某个 threshold，比如一分钟，那么继续从 leader db 里读取
         - 上面个两种方式增加了 leader 的负担，可以选取一个 replica whose last update is close to leader

      2. **Monotonic reads**： After users have seen the data at one point in time, they shouldn’t later see the data from some earlier point in time. （不想做时光机）

         固定用户对应固定的 replica 有个 map 方式

      3. **Consistent prefix reads**：Users should see the data in a state that makes causal sense: for example, seeing a question and its reply in the correct order.

         通常在 partitioned db 里会有这个问题

- **解释下 multi-leader based replication 比起 single-leader based replication 适用的场景? （2 点）**

      1. 多个 data center，网络延迟很大，这种 case 下，multi-leader 结构性能更佳
      2. 需要线下业务数据库支持，比如手机上的日历表，账本等等。现在很多软件都是手机端支持离线，云端还有数据库。

- **阐述下 multi-leader based replication 的几种可能解决 write conflit 的方法? (2 点）**

      1. 简单粗暴：avoid conflict，所有碰到同一个 record 的操作归给指定的 leader 操作，这样牺牲了 performance
      2. converging to a conistent state
         - 给每个 write 一个 unique ID， pick the write with the highest ID wins, and discard other writes. 缺点：造成了数据丢失
         - 给每一个 replica 一个 unique ID， 让发生在高序位的 replica always wins，跟上面一样
         - Somehow sort & merge the values ?
         - Record the conflit data and details, like when, happened at which table, 开发一个独立的辅助 application 来 handle conflict

      理想情况下，最后一个方式是最好的，但毕竟还是希望数据库自身能够处理 conflict，可惜，现在数据库没啥太好办法。

- **解释下 multi-leader 的三种拓跋图以及优劣?**

      环装，星型，以及 all-to-all，优劣略过，太 simple

---

## Leaderless replication 问题集合

- **举例几个 leaderless replication 的数据库**

      leaderless 数据库火于 dynamo db，受 dynamo db 影响，Riak， Cassandra and Voldemort 也都算。

- **解释下 read repair 跟 anti-entropy process?**

      Read repair：When a client makes a read from several nodes in parallel, it can detect any stale responses. For example, user 2345 gets a version 6 value from replica 3 and a version 7 value from replicas 1 and 2. The client sees that replica 3 has a stale value and writes the newer value back to that replica. This approach works well for values that are frequently read. 我靠，这又读还去更改数据库，做的有点多。

      Anti-entropy process：In addition, some datastores have a background process that constantly looks for differences in the data between replicas and copies any missing data from one replica to another. Unlike the replication log in leader-based replication, this _anti-entropy process_ does not copy writes in any particular order, and there may be a significant delay before data is copied. 也就是说白了不需要**replication log 额**

- **Leaderless 结构如何保证 write & read successful?** -> **QUORUMS FOR READING AND WRITING**

      讲真的，要知道 write successful 必须靠之后的 query 才能知道。 公式 W+R > N

- **Leaderless 结构如何检查 concurrent write 跟处理 conflict?**

      没有太好的办法，Cassandra 用了 LWW（Latest Write Win）的方式，所以在 Cassandra 里面也是推荐用 UUID 作为 Key。

---

## 练习

1. 思考如何设计 Google docs？
2. 思考如何设计一款 calendar，同时支持云端跟移动端业务，在移动端离线状态下，用户依然可以更新日历表。
