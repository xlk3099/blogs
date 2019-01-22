---
title: "Designing Data-Intensive Applications 6: Partition"
date: 2019-01-22T10:50:25+08:00
draft: true
---



还是那个思路，解读的时候

先了解

* 痛点
* 是啥
* 有几种模式
* pros and cons

现实中，处理数据库相关操作的时候，我们会遇到这些问题：

 	1. database or hardward  may fail at anytime
 	2. application may crash
 	3. interruption in the network unexpectedly cut off the application from the database, or one database from another
 	4. Several clients may write to the database at the same time, overwriting each other's changes
 	5. A client may read data that does not make sense because it has only partially been updated.
 	6. Race conditions between clients can cause surprising bugs.



这里讲的不是如何去减少这些问题，而是如何保证在有这些问题的情况下，读写仍然正确。

Transaction：is a way for an application to group several reads and writes together into a logical unit. 也就是说，all the reads & writes in a trasaction are executed as one operation. 要么全部成功，or 都失败（abort，rollback), if it fails, the application can safely retry. 

With transaction : error handling in application side is easier: we do not need to care about partial failure.

? How do you figure out whether we need transactions? first let's understand what safety guarantees transactions can provide and what costs are associated with them. 



Transaction was origined almost 45 years back in 1975 by IBM System R, the first SQL database.



In the late 2000s, nonrelational databases started gaining popularity. They aimed to improve upon relational status by offering a choice of a new data models, and by includinng replication and partitioning by default, but not **transaction**. Poor transaction was the only casualty of this movement.



With the hype around this new crop of distributed databases, there emerged a popular belief that transactions were the antithesis of scalability. aka, transaction actually affect the performance of scalability.



let's understand what transactions can do and its limitations first.



**The meaning of ACID**

safety guaranntees provided by transactions also known as ACID ( Atomicity, Consistency, Isolation, Durability) **Note:** The high-level is sound, but the devil is in the details, today when a system claims to be ACID compliant, it's unclear what guarantees you can actually expect, ACID has unfortunately become mostly a marketinng term.

BASE: for software which do not meet ACID criteria, basically availale, soft state, and eventual consistency.



**Atomicity**

简单来说就是，数据库不接受partial failure，要么全部成功，要么全部失败。upon a group of write requests, if any of them fails due to whatever reason, this transaction is aborted and the database will discard or undo any writes it has made so far in the transaction.

*abortability* might be a better word actually.

**Consistency**

作者吐槽了下consistency 被滥用了 哈哈。

In the context of ACID, consistency refers to an application-specific notion of the database being good state. 也就是说用来描述数据库状态。。。无语

what people expect? you have certain statements about the data that must always be true. Actually in this case, it's the responsibility of the application instead of the database.

Atomicity, isolation an durability are properties of the database, whereas consistency is a property of the application. 

**Isolation**

Most databases are accessed by several clients at the same time. 所以啊 level db 跟rocks db 真的是作弊啊。。。单进程。。

Isolation in the context of ACID means that concurrently executing transactions are isolationed from each other, they can not step on each other's toes.

**Durability**

Durability is the promise that once a transactions has commiteted successfully, any data it has written will not be forgotten, even if there is a hardware fault or the database crashes. 要等的就是这个承诺！

在single node database 里，可以通过non-volatile storage such as a hard drive or SSD, WAL(writing ahead log), 所以就算系统崩了也允许recovery。but in a replicated database, durability may mean that the data has been successfully copied to some number of nodes. in order to provide durability gurarantee, a database must wait until these writes or replications are complete before reporting a transaction as successfully commiteed. of course, there is no perfect durability, if all the hard dissks and all the backups are destroyed, there's no way the database can save you.

**replication and durability**

The truth is, nothing is perfect:

- If you write to disk and the machine dies, even though your data isn’t lost, it is inaccessible until you either fix the machine or transfer the disk to another machine. Replicated systems can remain available.
- A correlated fault—a power outage or a bug that crashes every node on a particular input—can knock out all replicas at once (see [“Reliability”](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch01.html#sec_introduction_reliability)), losing any data that is only in memory. Writing to disk is therefore still relevant for in-memory databases.
- In an asynchronously replicated system, recent writes may be lost when the leader becomes unavailable (see [“Handling Node Outages”](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch05.html#sec_replication_failover)).
- When the power is suddenly cut, SSDs in particular have been shown to sometimes violate the guarantees they are supposed to provide: even `fsync` isn’t guaranteed to work correctly [[12](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html#Zheng2013up)]. Disk firmware can have bugs, just like any other kind of software [[13](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html#Denness2015tz), [14](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html#Surak2015tz)].
- Subtle interactions between the storage engine and the filesystem implementation can lead to bugs that are hard to track down, and may cause files on disk to be corrupted after a crash [[15](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html#Pillai2014vx_ch7), [16](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html#Siebenmann2016ua)].
- Data on disk can gradually become corrupted without this being detected [[17](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html#Bairavasundaram2008vx)]. If data has been corrupted for some time, replicas and recent backups may also be corrupted. In this case, you will need to try to restore the data from a historical backup.
- One study of SSDs found that between 30% and 80% of drives develop at least one bad block during the first four years of operation [[18](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html#Schroeder2016us)]. Magnetic hard drives have a lower rate of bad sectors, but a higher rate of complete failure than SSDs.
- If an SSD is disconnected from power, it can start losing data within a few weeks, depending on the temperature [[19](https://learning.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch07.html#Allison2015ta)].

ACID Transactions 也就是说transaction 里面就一个操作，跟多个操作。

1. Single object and multiobject operations
2. Single object writes 可额能遇到的问题
   1. If the network connection is interrupted after the first 10 KB have been sent, does the database store that unparseable 10 KB fragment of JSON?
   2. If the power fails while the database is in the middle of overwriting the previous value on disk, do you end up with the old and new values spliced together?
   3. If another client reads that document while the write is in progress, will it see a partially updated value?

The need for multi-object transactions.

many distributed datastores have abandoned multi-object transactions because they are difficult to implement across partitions and they can get in the way in some scenarios where very high availability or performance is required.  如何实现distribubted transactions will be discussed in chapter 9.

1. SQL DB 为啥需要？
2. NoSQL DB 为啥需要？
3. In database with secondary indexes, 



**Handling errors and aborts**

ORM 一个新的缺点 , 吐槽了ORM 模型：Rails's ActiveRecord and Django dont retry aborted transactions

Although retrying an aborted transaction is a simple and effective error handling mechanism, it isnt

 perfect. 

1. if the transaction actually succeeded, but the network failed while the server tried to acknowledge the successful commit to the client(so the client thinks if tailed), double transaction might happen in this case. 
2. If the error is due to overload, retry will make the problem worse, not better. To avoid such feedback cycles, we need to limit the number of retries, and handle overloaded related errors different from other errors(if possible)
3. It's only worth retrying after transient errors (for example due to deadlock, isolation violation, temporary network interruptions, and failover); after a permanent error(e.g., constraint violation), a retry would be pointless
4. if the transaction also has side effects outside of the database, those side effects may happen even if transaction is aborted. e.g. if sending an email, you would not want to send the email again every time you retry the transaction. To make sure several different systems either commit or abort together, two-phase commit can help.
5. **If the client process fails while retrying, any data it was trying to write to the database is lost**



**Weak isolation levels**

concurrency bugs are hard to find by testing, because such bugs are only triggered when you get unlucky with the timing. 



??? what is serializable isolation, why it's related to weak isolation levels. 

接下下dirty read 跟dirty write。

**Read commited**

when reading from the database, you will only see data that has been committed (no dirty reads)

when writing to the database, you will only overwrite data that has been committed ( no dirty writes)

1. Dirty read: if a transaction has writen some data to the database, but the transaction has not yet committed or aborted, if another transaction can see the uncommited data, then it's called dirty read.

2. Dirty write: what happens if two transactions concurrently try to update the same object in a database? 

   normal case we expect the later write overwrites the earlier write. But there is a case: the earlier write is part of a transaction that has not yet commited, so the later write overwrites an uncommited write. 

4. how to implement read committed: 

   1. most commonly prevent dirty writes: using row-level locks, when a transaction wants to modify a particular object(row or document), it must first acquire a lock on that object. It must then hold that lock until the transaction is commited or aborted. Only one transaction can hold the lock for any given object.
   2. How to prevent dirty reads. One option is to use the same lock, this can ensure that a read could not happend while an object has a dirty, uncommited value.  实际上，requiring read locks does not work well in practice, because one long-running write transaction can force many read-only transactions to wait until the long-running transaction has completed. this harms the response time of read-only transaction and is bad for operability.  therefore, a practical way is for every object that is written, the database remembers both the **old commiteed value** and **the new value set by the transaction** that currently holds the writ lock.

   但是read commited 不能很好的解决race condition between two counter increments. 

   transaction 1:  get counter (42) -> set counter(43)

   transaction 2: get counter(42) -> set counter (43)

   the thing here is transaction 2 should set counter 44.

   read commited isolation 里面还有可能看到read skew 或者说non repeatable read 

**Snapshot isolation & Repeatable read**

长时间的DB process 比如

1. Back ups

1. Analytic queries and integrity checks

Read skew or non-repeatable read in most cases is acceptable but not for long DB process like back ups or analytic queries and integrity checks.

也就是说，在进行backup操作的时候，read commited isolation level non-repeatable read 是不可接受的。

snapshot isolation is the most common solution to this problem: The idea is that each transaction reads from a consistent snapshot of the database - that is , the transaction sees all the data that was commited in the database at the start of the transaction. Even if the data is subsequently changed by another transaction, each transaction sees only the old data from that particular point in time.  保证了in a same transaction, the same read returns same result. reapeatble read.

实现： 

1. 跟read commited 一样，用lock 来prevent dirty writes。
2. 跟read commited 类似，snapshot isolation 也是使用了多个version。 if a database only need to provide read commited isolation, but not snapshot isolation, it would be sufficient to keep two versions of an object, the commited version and the overwritten-but-not-yet-commited version. A typical approach is that read commited uses a separate snapshot for each query, while snapshot isolation uses the same snaphost for an entire transaction. 

**Visibility rules for observing a consistent snapshot**

**Indexes and snapshot isolation** 

问题：how do indexes work in a multi-version database? one option is to have the index simply point to all versions of an object and require an index query to filter out any object versions that are not visisble to the current transaction. When garbage collection removes old object versions that are no longer visisble to any transaction, the corresponding index entries can also be removed. 

... 略过有胆复杂

**Preventing lost updates**

描述下lost update 的问题

The lost update problem can occur if an application reads some value from the database, modifies it, and writes back the modified value (a *read-modify-write cycle*). If two transactions do this concurrently, one of the modifications can be lost, because the second write does not include the first modification. 

- Incrementing a counter or updating an account balance (requires reading the current value, calculating the new value, and writing back the updated value)
- Making a local change to a complex value, e.g., adding an element to a list within a JSON document (requires parsing the document, making the change, and writing back the modified document)
- Two users editing a wiki page at the same time, where each user saves their changes by sending the entire page contents to the server, overwriting whatever is currently in the database

**解决措施**

1. atomic operations provided by database:通常情况下 update a record, we follow read-modify-write cycles in application code, but luckily, many databases provide atomic update operations. 

   e.g. in mysql 

   ```mysql
   UPDATE COUNTERS SET VALUE = VALUE+1 WHERE KEY ='FOO
   ```

   注意ORM 模型又一个缺点，ORM 非常容易采用read-modify-write cycle instead of using atomic operations provided by database. **It belongs to a single-object operations**.

2. explicit locking: if db does not provide the atomic operations, then the application itself can do an explicit lock on the objects that are going to be updated, then the application can perform a read-modify-write cycle. in this case if any other transactions tries to concurrently read the same object, it is forced to wait until the first read-modify-write cycle has completed. 何时用这个？举个例子：consider a multiplayer game in which several player can move the same figure concurrently, 这种情况下 an atomic operation might not be sufficient.

3. **atomatically detecing lost updates**: 上面两个方式 是防止lost udpates, but think in another way, if the transaction manager detects a lost update, abort the transaction and force it to retry its read-modify-write cycle.  In fact, this is a very efficient way. PostgreSQL and Oracles provides this feature, but not MySQL. This is a great feature, because it does not require application code to use any special database featrues- you may forget to use a lock or an atomic operation and thus introduce a bug, but lost update  detection happens automatically and is thus less error-prone.
4. **conflict resolution and replication**: in replicated databases, preventing lost updates takes on another dimension: since they have copies of the data on multiple nodes, and the data can potentially be modified concurrently on different nodes. **atomic operations** can work well in a replicated context. 



**Write Skew**

characterizing write skew

1. because two transactions are updating two different objects.  Write skew can occur, if two transactions read the same objects, and then update some of those objects. 

   atomic single-object operations dont help, as multiple objects are invovled. 

2. The automatica detection of lost updates do not help here either.

3. Some databbases allow you to configure constraints, which are then enforecd by the database, but most databases do not have built-in support for such constratins, you may be able to implement them with triggers or materialized views, depending on the database, though.

4. If can use a serializable isolation level, the second-best option in this case is probably to explictly lock the rows that the transaction depends on.

more examples of write skew:

1. Meeting room booking systems: there can be two booking for the same meeting room at the same time.
2. multiplayter game.
3. claiming a user name
4. preventing double - spending.

**Serializability: 串行化**

Serializable isolation is usually regarded as the strongest isolation level. It guarantees that even though transactions may execute in parallel, the end result is the same as if they had executed one at a time, serially without any concurrency. 顾名思义，串行！in other words, the database prevents all possible race conditions. 

**How to implement serializability**

1. literally executing transactions in a serial order
2. Two phase locking (2PL) which used for several decades.
3. Optimistic concurrency control techniques, 乐观锁？

**Actual serial execution**

The simplest way of avoiding concurrency problems is to remove the concurrency entirely: to execute only one transaction at a time, in serial order, on a single thread.  尽管这个idea看上去很蠢，但是直到2007年才被可行。

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

the big downside of two-phase locking and the reason why it hasnot been used by everybody since the 1970s, is performance: transaction throughput and response times of quereis  are significantly worse under  2PL than under weak isolation.

2PL reduces a lot of concurrency and creates a lot locks(overhead)

Under 2PL, deadlocks can happen much more frequently.



**Predicate Locks**

解释下phantoms：one transaction changing the results of another transactions's search query. 

**Pessmistic vs Optimistic concurrency controls**

Pessmistic : Serial execution 

Optimistic: SSI (serializable snapshot isolation)



**Decisions Based On An Outdated Premise**

dirty read

dirty write

Non-repeatable read & read skew: in a transaction, it reads a record in the beginning and at the end, it read it again, the reuslts may be different.





race conditions

1. dirty writes
2. lost updates
3. write skew: because two transactions are updating two different objects.  Write skew can occur, if two transactions read the same objects, and then update some of those objects. 