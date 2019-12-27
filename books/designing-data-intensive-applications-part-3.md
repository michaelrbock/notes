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

* If the transaction actually succeeded, but the network failed...
