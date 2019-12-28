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

TODO
