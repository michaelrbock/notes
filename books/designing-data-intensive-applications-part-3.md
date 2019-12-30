# Designing Data-Intensive Applications - Part III

[Goodreads](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications)

-

## Chapter 7: Transactions

Many things can go wrong in data systems:

* Db software/hardware could fail at any time (including in the middle of a write).
* The application could crash (including halfway through a series of operations).
* Network interuptions.
* Multiple clients overwrite each other.
* Other race conditions.

*Transactions* group several reads and writes together and are executed as one operation: either the entire transaction succeeds (*commit*) or it fails (*abort, rollback*). If it fails, the app can safely retry.

Transactions provide *safety guarantees*: the application is free to ignore certain potential error scenarios and concurrency issues.

Sometimes, it's worth weakening/abandoning transactional guarantees (e.g. for higher performance/availability).

### ACID

ACID describes the safety guarantees provided by transactions.

Not all ACID implementations are equivalent, and the devil is in the details. *BASE (Basically Available, Soft state, Eventual consistency)* is a vague term that basically just means "not ACID".

#### Atomicity

When making several writes grouped into an *atomic* transaction, if the transaction cannot be commit due to a fault, then the transaction must be aborted and the db must discard/undo any writes made so far.

* Without atomicity, it's difficult to know which changes took effect and which didn't if an error occurs partway through. 
* Atomicity also allows us to safely retry.

#### Consistency

*Consistency* means that you have certain *invariants* about your data that must always be true. If a transaction starts in a valid state, a consistent transaction preserves that validity.

* Your db can *help* you enforce those invariants, but it is the job of the application to define and enforce what data is valid and invalid.
* A, I and D are properties of the database, but C is a property of the application.

#### Isloation

Concurrently executing transactions are *isolated* from each other: e.g. each transaction can pretend it is the only transaction running on the database. This helps avoid race conditions.

#### Durability

*Durability* is the promise that once a transaction has committed successfully, any data it has written will not be forgotten.

* In single-node, this means writing to disk and a write-ahead log.
* In replicated db, this means the data has been sucessfully copied to some number of nodes.

### Single-Object and Multi-Object Operations

ACID's *Atomicity* and *Isolation* often assume you want to modify several objects at once:  *Multi-object transactions* are needed to keep several pieces of data in sync.

* Multi-object transactions require a way of determining which read an write operations belong to the same transaction.
* Many nonrelational databases don't have a way of grouping operations, even in *multi-put* operations.

#### Single-object writes

Atomicity and isolation also apply when a single object is being changed, e.g. imagine a fault while writing/updating a 20 KB JSON document.

* Almost all storage engines provide atomicity and isolation on the level of a single object on one node.
* Atomicity is implemented using a log for crash recovery.
* Isolation can be implemented using a lock on each object so only one thread can access an object at a time.

#### The needs for multi-object transactions

* Relational DBs with foreign keys (or graph DBs with edges): references need to remain valid when inserting multiple records that refer to each other.
* Document DBs with denormalized data across objects.
* Secondary indexes that need to be updated when you change a value.

#### Handling errors and aborts

Unlike ACID, leaderless replication datastores work much more on a "best effort" basis: "the database will do as much as it can, and if it runs into an error, it won't undo something it has already done". It's the application's responsibility to recover from errors.

What could go wrong with retrying an aborted transaction:

* If the transaction actually succeeded, but the network failed, then retryig causes it to be performed twice without application-level dedupe.
* If the error is due to overload, retrying will make the problem worse. To avoid this, limit retries or use exponential backoff.
* It is only worth retrying after transient errors, not permanent errors (e.g. constraint violations).
* If the transaction has side effects outside the db, must be careful the systems commit or abort together.

### Weak Isolation Levels

Two transactions could lead to concurrency issues (race conditions) only if they touch the same data. Database transaction *isolation* tries to hide concurrency issues from the application.

* *Serializable* isolation means the database guarantees that transactions have the same effect as if they ran *serially*.
    * Serializeable isolation has a performance cost, and not all databases implement it.
    * Many systems use weaker (nonserializable) levels of isolation, which protect against *some* concurrency issues, but not all.

#### Read committed

The most basic (and a very popular) level of transaction isolation.

1. No *dirty reads*: when reading, you will only see data that has been committed.
1. No *dirty writes*: when writing, you will only overwrite data that has been committed.

**No dirty reads**

Prevents:

* Another transaction sees some object updates, but not others. Ex: see unread email, but unread count is not yet updated.
* A transaction seeing a write that is later rolled back.

**No dirty writes**

A *dirty write* is when an uncommitted value from part of an earlier transaction is overwritten by a write from a later transaction. Read committed must delay the second write until the first write's transaction is committed/aborted.

* Prevents bad outcomes when transactions update multiple objects.
* Does *not* prevent race condition between two counter increments.

**Implementing read committed**

* Dirty writes are prevented by using locks: a transaction holds a lock from start to commit/abort for each object that is modified.
    * Only one transaction can hold the lock for any object.
* Dirty reads are usually prevented not with locks, but instead by returning the old committed value to other transactions that read the object.

#### Snapshot Isolation and Repeatable Read

*Read skew* (*nonrepeatable read*): one transaction reads data from before and after another transaction. Read skew is considered acceptable under read committed isolation.

Situations that cannot tolerate temporary inconsistency:

* Backups.
* Analytic queries and periodic integrity checks.

*Snapshot isolation* is the solution: each transaction reads from a *consistent snapshot* of the db: the transaction sees all the data that was comitted in the db at the start of the transaction.

**Implementing snapshot isolation**

Write locks prevent dirty writes, like in read committed isolation.

* Principle of snapshot isolation: *readers never block writers, and writers never block readers*.
    * Allowing db to handle long-running read queries on a consistent snapshot at the same time as processing writes normally.

To implement: the db keeps several different committed versions of an object. This is called *multi-version concurrency control (MVCC)*.

* When a transaction is started, it is given a unique, always-incresing transaction ID.
* Each row has a `created_by` and (an initially empty) `deleted_by` field.
* Garbage collection removes rows that no transaction can access any longer.

**Visibility rules for observing a consistent snapshot**

Transaction IDs are used to decide which objects are not visible:

* No writes from any other in-progress transactions.
* No writes by aborted transactions.
* No writes made by transactions with a later ID.

Objects are visible if:

* At the time the reader's transaction started, the transaction that created the object had already comitted.
* The object is not marked for deletion by a comitted transaction.

**Indexes and snapshot isolation**

How do indexes work in multi-version db?

* Have index point to all versions and requery index query to filter out versions that are not visible to current transaction.
* Use an *append-only/copy-on-write* B-Tree variant.

**Naming confusion**

Unfortunately, snapshot isolation is often called *repeatable read* (or even *serializable*).

#### Preventing Lost Updates

The *lost update problem* can occur when an app reads some value from the db, modifies it, and writes back the modified value (a *read-modify-write cycle*). 

* If two transactions do this concurrently, one of the modifications can be lost.
    * Because the second write does not include the first modification.
    * The later write *clobbers* the earlier write.

Examples:

* Incrementing a counter or updating an account balance.
* Making a change to a complex value, e.g. JSON document (have to parse document, make change, write back).
* Two users editing a wiki page.

**Atomic write operations**

Many databases provide atomic update operations, e.g. `UPDATE counters SET value = value + 1 WHERE key = 'foo';`.

Atomic operationsa are implemented by taking an exclusive lock on the object: aka *cursor stability*.

**Explicit locking**

Application explicitly locks objects that are going to be updated. For example if there is logic that cannot be expressed as a db query.

**Automatically detecting lost updates**

Allow read-modify-write cycles to execute in parallel and, if the transaction manager detects a lost update, abort the transaction and force it to retry. DBs can perform this check efficiently in conjunction with snapshot isolation.

**Compare-and-set**

In dbs that don't provide transactions, there is sometimes a compare-and-set operation. It avoids lost updates by allowing an update to happen only if the value has not changed since you last read it. if the value doesn't match, the update has no effect, and the read-modify-write cycle must be retried.

**Conflict resolution and replication**

Locks and compare-and-set operations assume that there is a single up-to-date copy of the data (on one node). This does not apply in multi-leader or leaderless replication dbs. Instead, replicated dbs often allow concurrent writes to create conflicting versions (*siblings*) of a value.

* Atomic operations work well in a replicated context if they are commutative (you can apply them in a different order on different replicas). E.g. incrementing a counter or adding to a set.
* *Last write wins (LWW)* conflict resolution is prone to lost updates.

#### Write Skew and Phantoms

*Write skew* is neither a dirty write nor a lost update because two transactions are updating two different objects, but it's definitely a race condition. Write skew is a generalization of the lost update problem. Write skew can occur if two transactions read the same objects, and then update some of those objects (different transactions may update different objects).

Many of the above tools won't work to prevent write skew. Either need:

* Serializable isolation.
* Complicated database constraint.
* Explicitly lock the rows that the transaction depends on.

Examples:

* Doctors taking themselves off on-call, when at least 1 has to always be on call.
* Meeting room booking system.
* Multiplayer game.
* Claiming a username.
* Preventing double-spending.

**Phantoms causing write skew**

Write skew follows this pattern:

1. A `SELECT` query checks whether some requirement is satisfied.
1. Application code decides how to continue based on result.
1. If OK, application makes a write.

But often the check is for the *absence* of rows matching some condition, and the write *adds* a row matching the same condition. When a write in one transaction changes the result of a search query in another transaction, it is called a *phantom*.

**Materializing conflicts**

If the problem of phantoms is that there is no object to which we can attach the locks, you can artificially introduce lock object to the db. E.g. create a row corresponding to a time period (e.g. 15 min) for a room for a meeting room booking system. This is called *materializing conflicts*.

It's hard/error-prone and ugly to let concurrency control mechanism leak into application. Therefore, should prefer to use serializable isolation level if possible.

#### Serializability

TODO
