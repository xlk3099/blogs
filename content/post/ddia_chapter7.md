---
title: "Designing Data-Intensive Applications 7: Transaction"
date: 2019-01-22T10:50:25+08:00
draft: false
---

# **Summary**

## Race condition scenarios

<span style="font-size:larger;">Dirty Reads</span>

One client reads another client’s writes before they have been committed. The read committed isolation level and stronger levels prevent dirty reads.

<span style="font-size:larger;">Dirty Writes</span>

One client overwrites data that another client has written, but not yet committed. Almost all transaction implementations prevent dirty writes.

Two transactions concurrently try to update the same object in a database, normaly we expect the later writes should overwrite the ealirer write. But it's possible that the earlier write is part of a transaction and has not commited yet, and the later write overwrites **an uncommited** value.

<span style="font-size:larger;">Read skew(nonrepeatable reads)</span>

A client sees different parts of the database at different points in time. This issue is most commonly prevented with snapshot isolation, which allows a transaction to read from a consistent snapshot at one point in time. It is usually implemented with _multi-version concurrency control_ (MVCC).

<span style="font-size:larger;">Lost Updates</span>

Two clients concurrently perform a **read-modify-write** cycle. One overwrites the other’s write without incorporating its changes, so data is lost. Some implementations of snapshot isolation prevent this anomaly automatically, while others require a manual lock (`SELECT FOR UPDATE`).

Lost updates 跟 Dirty Writes 需要区分下。

<span style="font-size:larger;">Write skew</span>

A transaction reads something, makes a decision based on the value it saw, and writes the decision to the database. However, by the time the write is made, the premise of the decision is no longer true. Only serializable isolation prevents this anomaly.

<span style="font-size:larger;">Phantom reads</span>

A transaction reads objects that match some search condition. Another client makes a write that affects the results of that search. Snapshot isolation prevents **straightforward phantom reads**, but phantoms in the context of write skew require special treatment, such as index-range locks.

Weak isolation levels protect against some of those anomalies but leave you, the application developer, to handle others manually (e.g., using explicit locking). Only serializable isolation protects against all of these issues.

## 3 ways to implement serializable transactions

1. Literally executing transactions in a serial order, representative: redis, cost: can only work on a single CPU core.
2. Two phase locking(2PL): 统治了几十年，但现在很多 application avoid using it， 因为性能问题, 悲观锁
3. SSI（serializable snapshot isolation）： 新贵，use an optimistic approach， 乐观锁

最后要提出的是，multi-object transactions 在 SQL database 里面用的比较多，在 K-V 里面比较少，现有很多 K-V 数据库考虑到性能问题，不支持 multi-object transactions

---

# 笔记区

## Why we need transaction

1. database or hardward may fail at anytime
2. application may crash
3. interruption in the network unexpectedly cut off the application from the database, or one database from another
4. Several clients may write to the database at the same time, overwriting each other's changes
5. A client may read data that does not make sense because it has only partially been updated
6. Race conditions between clients can cause surprising bugs

## What is transaction in database systems

Transaction：is a way for an application to group several reads and writes together into a logical unit. 也就是说，all the reads & writes in a trasaction are executed as one operation. 要么全部成功，or 都失败（abort，rollback), if it fails, the application can safely retry. 这里讲的不是如何去减少这些问题，而是如何保证在有这些问题的情况下，读写仍然正确。

With transaction,error handling in application side is easier, we do not need to care about partial failure.

## When to use transactions

Understand what safety guarantees transactions can provide and the costs are associated with them.

<span style="font-size:larger;">**ACID**</span>

safety guaranntees provided by transactions also known as ACID ( Atomicity, Consistency, Isolation, Durability) **Note:** The high-level is sound, but the devil is in the details, today when a system claims to be ACID compliant, it's unclear what guarantees you can actually expect, ACID has unfortunately become mostly a marketinng term.

Atomicity, isolation an durability are properties of the database, whereas consistency is a property of the application.

**BASE**: for software which do not meet ACID criteria, basically availale, soft state, and eventual consistency.

<span style="font-size:larger;"> Atomicity</span>

简单来说就是，数据库不接受 partial failure，要么全部成功，要么全部失败。upon a group of write requests, if any of them fails due to whatever reason, this transaction is aborted and the database will discard or undo any writes it has made so far in the transaction.

_abortability_ [by author] might be a better word actually.

<span style="font-size:larger;"> Consistency</span>

作者吐槽了下 consistency 被严重滥用了 哈哈。

In the context of ACID, consistency refers to an application-specific notion of the database being good state. 也就是说用来描述数据库状态。

what people expect? you have certain statements about the data that must always be true. Actually in this case, it's the responsibility of the application instead of the database.

<span style="font-size:larger;"> Isolation</span>

Most databases are accessed by several clients at the same time. 又让我想起了很多 KV 数据库只支持单个 client，比如 LevelDB， RocksDB.

Isolation in the context of ACID means that concurrently executing transactions are isolationed from each other, they can not step on each other's toes. 没啥太多好解释的。

<span style="font-size:larger;"> Durability</span>

Durability is the promise that once a transactions has commiteted successfully, any data it has written will not be forgotten, **even if there is a hardware fault or the database crashes**. 要等的就是这个承诺！

在 single node database 里，可以通过 non-volatile storage such as a hard drive or SSD, WAL(writing ahead log) - 就觉得应该是 WAL, 所以就算系统崩了也允许 recovery。but in a replicated database, durability may mean that the data has been successfully copied to some number of nodes. in order to provide durability gurarantee, a database must wait until these writes or replications are complete before reporting a transaction as successfully commiteed.

**Note** There is no perfect durability, if all the hard dissks and all the backups are destroyed, there's no way the database can save you.

<span style="font-size:larger;"> Single object and multiobject operations</span>

<span style="font-size:larger;"> Single object operations</span>

虽然 atomicity 第一反应是觉得同时处理多个 objects，但对 single object 也是需要支持。甚至很多 KV 数据库只支持 single object 的 ACID.
比如假设向数据库写一个 20KB 的 JSON 文档。

1. If the network connection is interrupted after the first 10 KB have been sent, does the database store that unparseable 10 KB fragment of JSON?
2. If the power fails while the database is in the middle of overwriting the previous value on disk, do you end up with the old and new values spliced together?
3. If another client reads that document while the write is in progress, will it see a partially updated value?

所以 storage engines 也会给 single object 提供 ACID 的支持。

<span style="font-size:larger;"> The need for multi-object transactions</span>

many distributed datastores have abandoned multi-object transactions because they are difficult to implement across partitions and they can get in the way in some scenarios where very high availability or performance is required.

1. SQL DB 为啥需要? ensure foreign keys to be correct and up to date.
2. NoSQL DB 为啥需要？document DB 通常 join 能力很弱，所以一般会存储额外的 documents 来记录 join 信息，这些 documents 也需要同时更新。
3. **In database with secondary indexes**？ 这块目前不是很懂。

<span style="font-size:larger;"> Handling errors and aborts</span>

用 transaction 方便的地方在于，每次 transaction 失败的时候，它会被 aborted，application 端可以很方便的处理错误然后 retry。
作者顺带又吐槽了 ORM 模型，并举例了 Rails's ActiveRecord and Django dont retry aborted transactions
Although retrying an aborted transaction is a simple and effective error handling mechanism, it isnt
perfect.
abort and retry 用起来很方便，但这种处理方式其实是有很大的弊端:

1. if the transaction actually succeeded, but the network failed while the server tried to acknowledge the successful commit to the client(so the client thinks if tailed), double transaction might happen in this case.
2. If the error is due to overload, retry will make the problem worse, not better. To avoid such feedback cycles, we need to limit the number of retries, and handle overloaded related errors different from other errors(if possible)
3. It's only worth retrying after transient errors (for example due to deadlock, isolation violation, temporary network interruptions, and failover); after a permanent error(e.g., constraint violation), a retry would be pointless
4. if the transaction also has side effects outside of the database, those side effects may happen even if transaction is aborted. e.g. if sending an email, you would not want to send the email again every time you retry the transaction. To make sure several different systems either commit or abort together, two-phase commit can help.
5. **If the client process fails while retrying, any data it was trying to write to the database is lost**

总结下，就是对错误处理不够精细化，容易给服务器造成 too much overload。

## 隔离级别

<span style="font-size:larger;"> Read commited </span>

最基本的隔离级别，是个数据库基本都得有。

when reading from the database, you will only see data that has been committed (no dirty reads)
when writing to the database, you will only overwrite data that has been committed ( no dirty writes)

1. Dirty read 的定义 见总结
2. Dirty write 的定义 见总结
3. how to implement read committed:

   1. prevent dirty writes: 加 row-level locks。
   2. prevent dirty reads: 跟上述一样加锁, 但很多时候加锁不是很好的选择，requiring read locks does not work well in practice, because one long-running write transaction can force many read-only transactions to wait until the long-running transaction has completed. this harms the response time of read-only transaction and is bad for operability. therefore, a practical way is for every object that is written, the database remembers both the **old commiteed value** and **the new value set by the transaction** that currently holds the writ lock. 其实就是 MVCC

   但是 read commited 不能很好的解决（幻读）

<span style="font-size:larger;"> Snapshot isolation & Repeatable read</span>

Read skew or non-repeatable read in most cases is acceptable but not for long DB process like **DB backups** or **analytic queries** and **integrity checks**.

也就是说，在进行 backup 操作的时候，read commited isolation level non-repeatable read 是不可接受的。

snapshot isolation is the most common solution to this problem: The idea is that each transaction reads from a consistent snapshot of the database - that is , the transaction sees all the data that was commited in the database at the start of the transaction. Even if the data is subsequently changed by another transaction, each transaction sees only the old data from that particular point in time. 保证了 in a same transaction, the same read returns same result. reapeatble read.

实现：

1. 跟 read commited 一样，用 lock 来 prevent dirty writes。
2. 跟 read commited 类似，snapshot isolation 也是使用了多个 version。 if a database only need to provide read commited isolation, but not snapshot isolation, it would be sufficient to keep two versions of an object, the commited version and the overwritten-but-not-yet-commited version. A typical approach is that read commited uses a separate snapshot for each query, while snapshot isolation uses the same snaphost for an entire transaction.

<span style="font-size:larger;"> Indexes and snapshot isolation</span>

问题：how do indexes work in a multi-version database? one option is to have the index simply point to all versions of an object and require an index query to filter out any object versions that are not visisble to the current transaction. When garbage collection removes old object versions that are no longer visisble to any transaction, the corresponding index entries can also be removed.

... 略过有点复杂

<span style="font-size:larger;"> Preventing lost updates </span>

描述下 lost update 的问题

The lost update problem can occur if an application reads some value from the database, modifies it, and writes back the modified value (a _read-modify-write cycle_). If two transactions do this concurrently, one of the modifications can be lost, because the second write does not include the first modification.

- Incrementing a counter or updating an account balance (requires reading the current value, calculating the new value, and writing back the updated value)
- Making a local change to a complex value, e.g., adding an element to a list within a JSON document (requires parsing the document, making the change, and writing back the modified document)
- Two users editing a wiki page at the same time, where each user saves their changes by sending the entire page contents to the server, overwriting whatever is currently in the database

**解决措施**

1. atomic operations provided by database:通常情况下 update a record, we follow read-modify-write cycles in application code, but luckily, many databases provide atomic update operations.

    e.g. in mysql

    ```mysql
    UPDATE COUNTERS SET VALUE = VALUE+1 WHERE KEY ='FOO
    ```

    注意 ORM 模型又一个缺点，ORM 非常容易采用 read-modify-write cycle instead of using atomic operations provided by database. **It belongs to a single-object operations**.

2. explicit locking: if db does not provide the atomic operations, then the application itself can do an explicit lock on the objects that are going to be updated, then the application can perform a read-modify-write cycle. in this case if any other transactions tries to concurrently read the same object, it is forced to wait until the first read-modify-write cycle has completed. 何时用这个？举个例子：consider a multiplayer game in which several player can move the same figure concurrently, 这种情况下 an atomic operation might not be sufficient.

3. **atomatically detecing lost updates**: 上面两个方式 是防止 lost udpates, but think in another way, if the transaction manager detects a lost update, abort the transaction and force it to retry its read-modify-write cycle. In fact, this is a very efficient way. PostgreSQL and Oracles provides this feature, but not MySQL. This is a great feature, because it does not require application code to use any special database featrues- you may forget to use a lock or an atomic operation and thus introduce a bug, but lost update detection happens automatically and is thus less error-prone.
4. **conflict resolution and replication**: in replicated databases, preventing lost updates takes on another dimension: since they have copies of the data on multiple nodes, and the data can potentially be modified concurrently on different nodes. **atomic operations** can work well in a replicated context.

<span style="font-size:larger;"> Write Skew </span>

characterizing write skew

1. because two transactions are updating two different objects. Write skew can occur, if two transactions read the same objects, and then update some of those objects.

    atomic single-object operations dont help, as multiple objects are invovled.

2. The automatica detection of lost updates do not help here either.

3. Some databbases allow you to configure constraints, which are then enforecd by the database, but most databases do not have built-in support for such constratins, you may be able to implement them with triggers or materialized views, depending on the database, though.

4. If can use a serializable isolation level, the second-best option in this case is probably to explictly lock the rows that the transaction depends on.

more examples of write skew:

1. Meeting room booking systems: there can be two booking for the same meeting room at the same time
2. multiplayter game
3. claiming a user name
4. preventing double - spending

Serializability

<span style="font-size:larger;">Serializability: 串行化</span>

Serializable isolation is usually regarded as the strongest isolation level. It guarantees that even though transactions may execute in parallel, the end result is the same as if they had executed one at a time, serially without any concurrency. 顾名思义，串行！in other words, the database prevents all possible race conditions.

**How to implement serializability**

1. literally executing transactions in a serial order
2. Two phase locking (2PL) which used for several decades.
3. Optimistic concurrency control techniques, 乐观锁？

**Actual serial execution**

The simplest way of avoiding concurrency problems is to remove the concurrency entirely: to execute only one transaction at a time, in serial order, on a single thread. 尽管这个 idea 看上去很蠢，但是直到 2007 年才被可行。

1. RAM became cheap enough that for many use cases it is now feasible to keep the entire active dataset in memory.
2. Database designers realized that OLTP transactions are usually short and only make a small number of reads and writes. By contract, long-running queries are typically read-only, so they can be run on a consistent snapshot outside of the serial execution loop.

This approach of executing transactions serially is implemented in VoltDB/H-Store, Redis and Datomic, in fact. A system designed for single-threaded execution can sometimes perform better than a system that supports concurrency, because it can **avoid the coordination overhead of locking.** However, its throughput is limited to that of a single CPU core.

**Summary of serial execution**

1. Every transaction must be small and fast, because it takes only one slow transaction to stall all transaction processing.
2. It is limited to use cases where the active dataset can fit in memory. Rarely accessed data counld potentially be moved to disk, but if needed to be accessed in a single-threaded transaction, the system would get very slow.
3. Write throughput must be low enough to be handled on a single CPU core, or else transactions need to be partitioned without requiring cross-parittion coordination.
4. Cross-parition transactions are possible, but there is a hard limit to the extent to which they can be used.

**Two-Phase Locking(2PL)**

The blocking of readers and writers is implemented by a having a lock on each object in the database. The lock can either be in shared mode or in exclusive mode. The lock is used as follows:

1. If a transaction wants to read an object, it must first acquire the lock in shared mode. Several transdactions are allowed to hold the lock in shared mode simultaneously, but if another transaction already has an exclusive lock on the object, theest transactions must wait.
2. If a transaction wants to write to an object, it must first acquire the lock in exclusive mode. No other transaction may hold the lock at the same time(either in sahred or in exclusive mode), so if there is any existing lock on the object. The transaction must wait.
3. After a transaction has accquired the lock, it must continue to hold the lock until the end of the transaction(commit or abort). This is where the name "two-phase" comes from: the first phase(while the transaction is executing) is when the locks are acquired, and the second phase (at the end of the transcation) is when all the locks are released.

Since so many locks are in use, it can happen quite easily that transaction A is stuck waiting for trasaction B to release its lock, and vice versa. This situation is called deadlock. The database automatically detects deadlocks between transdactions and aborts one of them so that the others can make progress. The aborted transaction needs to be retried by the application.

**Performance of the 2PL**

the big downside of two-phase locking and the reason why it hasnot been used by everybody since the 1970s, is performance: transaction throughput and response times of quereis are significantly worse under 2PL than under weak isolation.

2PL reduces a lot of concurrency and creates a lot locks(overhead)

Under 2PL, deadlocks can happen much more frequently.

**几种不同的锁 了解下**

1. Shared Locks
2. Exclusive Locks
3. Predicate Locks
4. Index-Range Locks