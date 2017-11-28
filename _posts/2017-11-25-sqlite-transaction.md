---
layout: post
title:  "Sqlite Transaction Review"
date:   2017-11-25 19:59:03 -0800
---

Recently I started reading the book [Designing Data-Intensive Applications](https://dataintensive.net/). This book has gained a lot of praises these days, It provides a pretty good overview of key distributed system concepts, including transactions, replication, partitioning and consensus. After finishing the **Transactions** chapter, I became curious about how the current database systems are actually implementing transactions. Sqlite, the simple yet widely used in our everyday life database, comes into my mind. In this post, I will share my learnings on Sqlite transactions.

## Sqlite Overview
Let's start with a Sqlite overview. Sqlite is small, transactional database yet supporting most SQL-92 syntaxes and transactions. It is widely used in embedded systems (phones, PDAs and smart devices). In Sqlite, databases are stored as operating system files.

One key distinction between Sqlite and other client/server database engines (PostgreSQL, MySQL, Oracle ..) is Sqlite does not have the client-server architecture. It is only a *library*. This caused a lot confusion on me. For example, how does Sqlite conduct concurrency control for multiple threads? After some research, it turned out Sqlite uses OS file locks, so that processes understand the locks set by other processes.

Another key concept related to transaction management is [Pager](https://github.com/mackyle/sqlite/blob/master/src/pager.c), Sqlite's caching and transaction manager. For Sqlite, data is managed at page level, and Pager is designed to do all page managements. Pager ensures ACID property of the transaction. It is aware of pages that are cached, added or modified, also, it controls the journaling of the old pages so that recovery can happen correctly during power loss. Before working on a page, Pager always fetches it to memory on demand.

## Transaction Implementation
Let's give a quick review about transaction *ACID* properties.

1. **A**tomicity: either all operations in the transaction are done, or none is done.
2. **C**onsistency: this is an application requirement. We will skip it.
3. **I**solation: Transactions can run independently without affecting each other. The most straightforward solution is serialization (transactions can be viewed as running sequentially).
4. **D**rability: The transaction outcome is persisted even with power loss or outages. Usually this is ensured by writing data to persistent storage.

In the following post we will focus on how Sqlite implements atomicity and isolation.

### Isolation Guarantee
Sqilte supports isolation by serializing the transactions (the transactions can be logically viewed as running serially). This is achieved with [**two-phase locking (2PL)**](https://en.wikipedia.org/wiki/Two-phase_locking), one of the widely used concurrency control algorithms. During 2PL, transactions are allowed to read the same object as long as no transactions are writing it. However, when a transaction wants to write an object, it must obtain exclusive access lock for the object. A write transaction will only release the lock when the transaction is committed or aborted.

To implement 2PL, Sqlite has to implement locks that are recognizable by different processes/threads. As we discussed before, Sqlite is just a library without a centralized manager. It use OS native file lock to implement different locks for both read and write access. In Sqlite, locks have several states. You can refer to [this](http://flylib.com/books/en/1.91.1.17/1/) for more details. The basic idea is Sqlite implements shared and exclusive locks to improve the read concurrency. Multiple read transactions can happen together, but to write DB, you have to grab the exclusive lock and no read lock is allowed during write.

To simplify implementation, Sqlite locks on the whole database files, rather than at  row- or table- level. This means that for each database, there can be at most one write transaction going on, although multiple read transaction can run without affecting each other.

As a side note, 2PL is widely used to ensure isolation by serializing the transactions. There are other ways to implement [*weaker* but *faster* isolation](https://dataintensive.net/), such as *read committed* and *snapshot isolation*. 2PL is also sometimes criticized due to the relatively low parallelism.

## Atomicity Guarantee
To ensure atomicity, Pager uses journals to record the operations to take. Once rollback is needed or power outage happens, Pager can correctly recover to the original state as if the transaction has never been executed before.

Sqlite organizes the data in logical pages. Before actually modifying pages during a transaction, it applies write-ahead logging (WAL) to dump the original page to the journal. The journal is done at page level, and there is a single journal per database.

Journals are initially kept in memory, and are flushed out before any real database changes. The flushing ensures that rollback can happen even with power failures. After the database happens, Pager will **finalize** the journal by either abandoning it or applying it to rollback transaction. A transaction is considered committed or aborted only after the journal is finalized.

Be cautious when using transaction Atomicity. Per Sqlite [documentations](https://sqlite.org/lang_transaction.html),

```
[During a transaction], For certain errors, SQLite attempts to undo just the one statement it was working on and leave changes from prior statements within the same transaction intact and continue with the transaction. However, depending on the statement being evaluated and the point at which the error occurs, it might be necessary for SQLite to rollback and cancel the entire transaction. An application can tell which course of action SQLite took by using the sqlite3_get_autocommit() C-language interface.
```

So, it can happen that a transaction is partially executed. One way to avoid this is checking the status of each statement during a transaction, and abort it if any statement goes wrong. An example can be found [here](http://www.askyb.com/cpp/c-sqlite-example-with-atomic-transaction/).

## Other Thoughts
Can Sqlite lock at logical page/row level instead of DB level? This is possible, but the implementation will be more complex. In fact,some database implementations even lock at per-row level. For Sqlite, there is no need to provide such level concurrency when most installations are on embedded systems. Here Sqlite chooses simplicity over concurrency.

## Summary
This post shares some learnings about how Sqlite implements transactions. As a summary, Sqlite uses Pager to manage transactions. OS File locks are applied to isolate the transactions, and simple journals are utilized for atomic execution.
