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

### Serializability

Serializable isolation is the strongest isolation level. It guarantees that even though transactions may execute in parallel, the end result is the same as if they had executed one at a time, *serially*, without any concurrency. The db prevents all race conditions.

#### Actual Serial Execution

The simplest way to avoid concurrency problems is to avoid concurrency: only execute one transaction at a time, in serial order, on a single thread.

* RAM is cheaper now.
* OLTP transactions are usually short and only make a small number of reads & writes. Long-running analytic queries are usually read-only and can be run on a consistent snapshot outside of the serial execution loop.

**Encapsulating transactions in stored procedures**

Systems with single-threaded serial transaction processing don't allow interactive multi-statement transactions. Throughput would be dreadful because the db would spend most of its time waiting for the application to issue the next query for the current transaction.

Instead, the application must submit the entire transaction code as a *stored procedure*. If all the required data is in memory, the stored procedure can execute very fast without waiting for network or disk I/O.

**Summary of serial execution**

* Every transaction must be small and fast.
* Limited to use cases where active dataset can fit in memory.
* Write throughput must be low.
* Cross-partition transactions are hard.

#### Two-Phase Locking (2PL)

Most widely-used algorithm for serializability in databases: *two-phase locking (2PL)*.

Several transactions are allowed to concurrently read the same object as long as nobody is writing to it. But as soon as anyone wants to write (modify or delete) an object, exclusive access is required:

* If transaction A has read an object and transaction B wants to write to that object, B must wait until A commits or aborts before continuing. (B can't change the object behind A's back.)
* If transaction A has written an object and transaction B wants to read that object, B must wait until A commits or aborts before it can continue.
    * Reading an old version of the object is not acceptable under 2PL.

Writers block other writers and readers and vice versa.

**Predicate locks**

A *predicate lock* belongs to all objects that match some search condition (e.g. a `WHERE` clause). Applies to even objects which do not yet exist in the database (phantoms).

**Index-range locks**

Predicate locks do not perform well, so most dbs implement *index-range locking* (aka *next-key locking*). They simplify a predicate by matching a greater set of objects, e.g. by attaching to an index.

#### Serializable Snapshot Isolation (SSI)

*Serializable snapshot isolation (SSI)* provides full serializability with only a small performance penalty.

**Pessimistic versus optimistic concurrency control**

Two-phase locking is *pessimistic* concurrency control: if anything might go wrong, it's better to wait until the situation is safe again. By contrast, SSI is *optimistic*: transactions continue instead of blocking, and when about to commit, the db checks if anything bad happened.

**Decisions based on an outdated premise**

Transactions often take action based on a *premise* (a fact that was true earlier in the transaction). But the result of the query many no longer be up-to-date by the time the transaction commits. In order to provide serializable isolation, the db must detect situations where there is a causal dependency between queries and writes in a transaction and abort.

The db must detect when a query result might have changed:

* Reads of a stale MVCC object version (uncommitted write occurred before the read).
* Writes that affect prior reads (the write occurs after the read).

## Chapter 8: The Trouble with Distributed Systems

### Faults and Partial Failures

Software on a single computer (if the hardware is working) is *deterministic*: the same operation always produces the same result.

In a distributed system, some parts of the system can be broken and others working, a *partial failure*. Partial failures are *nondeterministic*.

#### Cloud Computing and Supercomputing

Supercomputers are used for computationally intensive scientific computing tasks and is more similar to a single-node computer: if any part of the system fails, just let everything crash and restart.

By contrast, most internet applications are *online*; they need to serve users with low latency at any time:

* Nodes are built from commodity machines.
* Network is based on IP and Ethernet.
* *Something* is likely always broken in a system with thousands of nodes.
* A system that can tolerate failed nodes is very useful.
* Geographically distributed.

You can build a realiable system from unreliable components.

### Unreliable Networks

The internet and most internal networks in datacenters are *asynchronous packet networks*. One node can send a message to another node. Many things can go wrong:

* The request may have been lost.
* Your request may be waiting in a queue.
* The remote node fails.
* The remote node has temporarily stopped responding. 
* The response is lost on the network.
* The response has been delayed.

If you send a request to another node and don't receive a response, it is *impossible* to tell why. The usual way of handling this issue is a *timeout*. When a timeout occurs, you still don't know whether the remote node got your request or not.

#### Network Faults in Practice

A *network fault* is also known as a *network partition* or *netsplit*.

You need to know how your system responds to network problems: define and test your error handling of network faults.

### Detecting Faults

Many systems need to automatically detect faulty nodes, e.g.:

* A load balancer needs to know to stop sending requests to a dead node.
* If the leader fails in a single-leader replication database, a follower needs to be promoted to new leader.

Feedback about a remote node being down is useful, but you can't rely on it. To be sure a request was successful, you need a positive response from the application itself. If something has gone wrong, you have to assume you may get no response and need to use a timeout.

#### Timeouts and Unbounded Delays

There is no simple answer to how long a timeout should be. Too long and you have to wait a long time for a node to be declared dead. Too short and you risk declaring a node dead when it is only temporarily slowed down.

Prematurely declaring a node dead is problematic: e.g. could perform an action twice if another node takes over.

When a node is dead, its responsibilities need to be transferred to other nodes. If already overloaded, this could lead to cascading failures.

**Network congestion and queueing**

The variability of packet delays on computer networks is most often due to queueing:

* *Network congestion* leads to queing on a network switch.
* OS queues incoming request if CPU cores are all busy.
* Virtual Machine monitor pauses OS while another VM uses a CPU core.
* TCP performs *flow control*.

When resources are shared (public clouds or multi-tenant datacenters) a *noisy neighbor* may make network delays highly variable.

Choose timeouts experimentally: measure network round-trip times over an extended period. Systems can continuously measure response times and variability (*jitter*) and automatically adjust timeouts.

#### Synchronous Versus Asynchronous Networks

Datacenter networks and the internet use packet switching, not *circuits* (a fixed guaranteed amount of bandwith, the way a telephone network works). The internet is optimized for *bursty* traffic.

The internet shares network bandwidth *dynamically*.

### Unreliable Clocks

Applications depend on clocks and time in various ways, e.g. (1-4: *durations*, 5-8: *points in time*):

1. Has the request timed out?
1. What's the 99th percentile response time of a service?
1. What's the QPS?
1. How long did a user spend on the site?
1. When was the article published?
1. When should the reminder email be sent?
1. When does the cache entry expire?
1. What is the timestamp on this log entry?

Networks have variable delays which makes it hard to determine the order things happened across multiple machines. Each machine has its own clock hardware device which is not perfectly accurate. Network Time Protocol (NTP) can be used to adjust computer clocks according to a group of servers that get their time from a more accurate source, like GPS.

#### Monotonic Versus Time-of-Day Clocks

**Time-of-day clocks**

A *time-of-day clock* returns the *wall-clock time*: the current time and date according to some calendar, e.g. `clock_gettime(CLOCK_REALTIME)` on Linux which returns the number of seconds since the *epoch* midnight UTC on Jan 1, 1970, not countind leap seconds.

Time-of-day clocks are usually synchronized with NTP. If it gets too far ahead, it may be forcibly reset and appear to jump backwards in time. These jumps and the fact that they ignore leap seconds make time-of-day clocks unsuitable for measuring elapsed time.

**Monotonic clocks**

A *monotonic clock* is suitable for measuring a duration, such as a timeout: `clock_gettime(CLOCK_MONOTONIC)` on Linux, for example. The name comes from the fact that they are guaranteed to always move forward.

You can check the value of the monotonic clock at one point in time, do something, and then check the clock again at a later time. The *difference* tells you how much time elapsed. However, the *absolute* value is meaningless and can't be compared across computers.

NTP adjusts the frequency at which the monotonic clock moves forward: *slewing* the clock if it detects the computer's local quartz is moving faster or slower than the NTP server.

#### Clock Synchronization and Accuracy

Hardware clocks and NTP aren't as reliable or accurate as one might hope:

* The quartz clock in a computer *drifts* (runs faster or slower than it should).
* When a computer's clock synchronizes with NTP, it may be forcibly reset and time jumps backwards or forwards.
* NTP has various problems as well.

#### Relying on Synchronized Clocks

If a clock is defective or NTP is misconfigured, the result is most likely to be subtle rather than a dramatic crash. But if the software requires synchronized clocks, you should carefully monitor clock offsets between machines and declare nodes dead whose clock drifts too far.

**Timestamps for ordering events**

Timestamps from time-of-day clocks are a dangerous way to order events. *Last write wins* (LWW) has many problems:

* A node with a lagging clock is unable to overwrite values written by a node with a fast clock.
* LWW cannot distinguish betweens sequential writes and truly concurrent writes. Version vectors must be used.
* Two nodes could generate the same timestamp.

*Logical clocks*, based on incrementing counters rather than quartz, are a safer alternative for ordering events than *physical clocks*.

**Clock readings have a confidence interval**

Even if you syncrhonize with an NTP server every minute, drift can be several milliseconds. With an NTP server on the public internet, the best possible accuracy is likely in the tens of milliseconds.

Thus, you can think of a clock reading as a range of times with a confidence interval. Google's Spanner Time API reports time as: `[earliest possible, latest possible]`.

**Synchronized clocks for global snapshots**

Creating monotonically increasing transaction IDs (for implementing snapshot isolation) in a distributed system is a hard problem.

Spanner compares timestamp intervals to determine causality: e.g. if `A = [Aearliest, Alatest] and B = [Bearliest, Blatest]` and `Aearliest < Alatest < Bearliest < Blatest`, then B definitely happened after A.

#### Process Pauses

Example: how does a node know that it's the leader in a single-leader per partition database? It obtains a *lease* from the other nodes (a lock with a timeout - only one node can hold at a time) and must periodically renew the lease. 

Reasons why a thread might be paused (for a long time):

* *Garbage collection* that stops all running threads.
* VMs can be *suspended* and *resumed*.
* A CPU spending time on other virtual machines is known as *steal time*.
* Application performs synchronous disk (or network filesystem) access.
* An OS may allow *swaping to disk (paging)*: memory access may result in a page fault that requires a page from disk to be loaded into memory. In extreme cases, the OS may spend most of its time swapping pages in and out of memory (*thrashing*).

Running threads are *preempted* and resumed later. On a single machine, we have tools for making things thread-safe. In a distributed system, a node may be paused, but the rest of the world continues moving.

**Response time guarantees**

A *hard real-time* system has a specified *deadline* by which the software must respond. Used for software in dangerous/physical environments, e.g. aircraft, robots, cars.

Providing real-time guarantees requires: a *real-time operating system* (RTOS) that allows processes to be scheduled with a guaranteed allocation of CPU time, library functions must document their worst-case execution times, dynamic memory allocation may be restricted or disallowed.

**Limiting the impact of garbage collection**

One idea is to treat GC pauses like planned outages of a node and let other nodes handle requests from clients while one node is garbage collecting.

### Knowledge, Truth, and Lies

A node in the network cannot *know* anything for sure - it can only make guesses based on the messages it receives/doesn't receive via the network.

In a distributed system, we can state the assumptions we are making about the behavior (the *system model*) and design the actual system to meet those assumptions and provide reliable behavior.

#### The Truth is Defined by the Majority

A node cannot necessarily trust its own "judgement". A distributed system cannot exclusively rely on a single node. Instead, many distributed systems rely on a *quorum*, voting among the nodes: decisions require some minimum number of votes from several nodes to reduce the dependence on any one node.

**The leader and the lock**

Frequently, a system requires there be only one of some thing, e.g.:

* One node is leader of a database partition.
* Only one transaction can hold the lock for a particular resource.
* One username per user.

Even if a node believes it is "the chosen one", that doesn't necessarily mean a quorum of nodes agrees. A node may have formerly been the leader, but the other nodes could have declared it dead in the meantime (e.g. due to a network interuption or GC pause)

If a node continues acting as the chosen one, even though its not, it could cause problems.

**Fencing tokens**

A *fencing token* is a number that increases (e.g. by the lock service like ZooKeeper) every time a lock is granted. Requests with a lower fencing token value are rejected.

#### Byzantine Faults

Fencing tokens can detect and block a node that is *inadvertntly* acting in error (e.g. because it hasn't yet found out its lease has expired).

We assume that nodes are unrealiable but honest: when a node responds, it is telling the "truth" to the best of its knowledge.

If there is a risk any nodes may "lie", such behavior is called a *Byzantine fault* and the problem of reaching consensus in this untrusting environment is known as the *Byzantine Generals Problem*.

A system is *Byzantine fault-tolerant* if it continues to operate correctly even if some nodes are malfunctioning, not obeying the protocol, or if malicious attackers are interfering:

* In aerospace environments, data in the computer may become corrupted by radiation.
* Some participants may attempt to defraud others, e.g. in the Bitcoin network, mutually untrusting parties agree whether a transaction happened without a central authority.

Byzantine fault-tolerant algorithms are complicated, and we assume that all nodes are controlled by the same organization. Web applications need to expect arbitrary/malicious behavior of clients, but don't need Byzantine fault tolerance because the server is the authority.

**Weak forms of lying**

There are pragmatic steps to take towards better reliability without full Byzantine fault tolerance:

* Checksums in application-level protocol.
* Santize user input. Prevent DDOS. Firewalls. Sanity checks.

#### System Model and Reality

Common system models for timing assumptions:

* *Synchronous model*: assumes *bounded* network delay, process pauses, and clock drift. Not realistic model of most systems.
* *Partially synchronous model*: behaves like a synchronous system *most of the time*, but sometimes exceeds bounds for network delay, process pauses, and clock drift. Realistic model of many systems.
* *Asynchronous model*: an algorithm is not allowed to make any timing assumptions and does not even have a clock. Very restrictive.

Common system models for nodes:

* *Crash-stop faults*: a node can fail only in one way: by crashing.
* *Crash-recovery faults*: nodes may crash and then start responding again after some time. Nodes have stable storage that is preserved across crashes.
* *Byzantine (arbitrary) faults*: described above.

The most useful models are partially synchronous and crash-recovery.

**Correctness of an algorithm & Safety and liveness**

To define what it means for an algorithm to be *correct*, we can describe its *properties*.

A *liveness* property often includes the word "eventually" in its description, e.g. *availability*. Liveness means *someting good eventually happens*.

A *safety* property means *nothing bad happens*, e.g. *uniquness* or *monotonic sequence*. If a safety property is violated, we can point at a particular point in time at which it was broken. For distributed system algorithms, it must *always* hold.

**Mapping system model to the real world**

Real implementations may have to include code to handle cases that we assumed impossible in theoretical description. But proving an algorithm correct does not mean its *implementation* on a real system will always behave correctly.

## Chapter 9: Consistency and Consensus

Like transactions, it is helpful to find general-purpose abstractions with useful guarantees that applications can rely on.

*Consensus*, getting all the nodes to agree on something, is one of the most important abstractions for distributed systems.

### Consistency Guarantees

Beacuse of replication lag (write requests arrive on different nodes at different times), two database nodes may have different data at the same time.

*Eventual consistency*, provided by most replicated databases, means that if you stopped writing, eventually all replicates would *converge* to the same value. This is a weak guarantee: it says nothing about *when* the replicas will converge.

Stronger consistency guarantees are easier to use correctly but may have worse performance or fault tolerance.

### Linearizability

*Linearizability* (aka *atomic consistency*, *strong consistency*, *immediate consistency*, *external consistency*) is the idea is to make a system appear as if there were only one copy of the data, you don't have to worry about replication lag, and all operations on it are atomic.

Linearizability is a *recency guarantee*: as soon as one client successfully completes a write, all clients reading must be able to see the most recent value.

#### What Makes a System Linearizable?

1. If one client's read returns a new value, all subsequent reads must also return the new value, even if the write operation has not yet completed.
1. Operations always move forward in time. Includes read, write, and compare-and-set operations.

#### Linearizability Versus Serializability

*Serializability* is an isolation property of *transactions* across multiple objects. It guarantees that transactions behave as if they were executed in *some* serial order.

*Linearizability* is a recency guarantee on reads and writes of an individual object and doesn't group operations together into transactions so it does not prevent write skew without taking additional measures such as materializing conflicts.

A db providing both is known as *strict serialzability* or *strong one-copy serializability*. Implementations of serializability based on 2PL or actual serial execution are typically linearizable. SSI is not linearizable by design.

#### Relying on Linearizability

**Locking and leader election**

Single-leader replication systems need to ensure there is only one leader. One way to elect a leader is to use a lock: every node that starts up tries to acquire the lock, and the one that succeeds becomes the leader. The lock must be linearizable so that all nodes agree. ZooKeeper and etcd do this.

**Constraints and uniqueness guarantees**

Uniqueness constraints are common in databases: e.g. a username or email for a user, a file path/name. Similar issues arise with bank account balances, selling items of limited stock, booking seats in an airplane/theater.

A hard uniqueness constraint requires linearizability.

**Cross-channel timing dependencies**

For example if a server stores an image in file storage and adds a message to a queue to tell an image resizer job to fetch the image and resize it, there could be a race condition. If the file storage is linearizable, this issue is avoided.

#### Implementing Linearizable Systems

Replication methods and whether they can be linearizable:

* *Single-leader replication*: *potentially linearizable* as long as you make reads from the leader or synchronously-updated followers and as long as you don't use snapshot isolation.
* *Consensus algorithms*: *linearizable*, e.g. ZooKeeper or etcd.
* *Multi-leader replication*: *not linearizable* because they concurrently process writes on multiple nodes and asynchronously replicate them to other nodes.
* *Leaderless replication*: *probably not linearizable* though sometimes claimed by requiring quorum reads and writes (w + r > n).

**Linearizability and quorums**

There is a race condition that causes nonlinearizable execution, even when using strict quorum in Dynamo-style systems. LWW also is not linearizable.

It *is* possible to make Dynamo-style quorums linearizable at the cost of reduced performance: a reader must perform read repair synchronously.

#### The Cost of Linearizability

A network interruption forces a choice between linearizability and availability.

**The CAP theorem**

Trade off:

* If your application *requires* linearizability and some replicas are disconnected due to a network problem, they become *unavailable*.
* If your application *does not require* linearizability, then each replica can process requests independently (e.g. multi-leader). The system can remain *available* in the face of a network problem, but is not linearizable.

Network partitions are inevitable, so CAP is really: *either Consistent or Available when Partitioned*.

**Linearizability and network delays**

Linearizability is a useful guarantee, but few systems are in practice, because of *performance*, not fault tolerance. Even RAM in a multi-core CPU is not linearizable(!) because each core has its own memory cache.

### Ordering Guarantees

The order in which things happen is an important fundamental theme.

#### Ordering and Causality

One reason ordering is important is *causality*, for example:

* There is a *causal dependency* between a question and the answer.
* In multi-leader replication, some operations depend on others, e.g. a row must be created before being updated.
* Reading from a consistent snapshot in Snapshot Isolation means *consistent with causality*: seeing the effects of all operations that happened causally before that point.
* SSI detects causal dependencies between transactions to prevent write skew.

A system that obeys the ordering imposed by causality is *causally consistent*.

**The causal order is not a total order**

*Linearizability*: In a linearizable system, we have a *total order* of operations. With a single copy of the data and atomic operations, we can always say which operations happened first.

*Causality*: Two events are ordered if they are causally related (one happened before the other), but they are incomparable if they are concurrent. Causality defines a *partial order*.

**Linearizability is stronger than causal consistency**

Linearizability *implies* causality but has a performance penalty. It is possible to provide causal consistency that does not slow down due to network delays and remains available in the face of network failures.

**Capturing causal dependencies**

Techniques include version vectors and those seen in SSI.

#### Sequence Number Ordering

We can use *sequence numbers* or *timestamps* (from a *logical clock*) to order events. They provide a *total order* that is *consistent with causality*. You can compare two sequence numbers to determine which operation happened later. This is easy to do in single-leader replication.

**Noncausal sequence number generators**

There are various ways to generate sequence numbers for operations that are *not consistent with causality*.

**Lamport timestamps**

Each node has a unqiue ID and keeps a counter of the number of operations it has processed. The *Lamport timestamp* is a pair of *(counter, node ID)*. This provides a total ordering.

Every node and client keeps track of the *maximum* counter value it has seen so far and includes that on every request.

**Timestamp ordering is not sufficient**

Nodes often have to decide on things *right now*, and Lamport timestamps/total ordering only works after the fact.

#### Total Order Broadcast

A single-leader replication can determine total order by sequencing all operations through a single CPU core on the leader. How to scale this or handle failover is known as *total order broadcast* or *atomic broadcast*.

Single leaders per-partition that only maintain ordering per-partition cannot offer consistency guarantees across partitions.

Total order broadcast is a protocol for exchanging messages between nodes that requires two safety properties:

* *Reliable delivery*: no messages are lost - if a message is delivered to one node, it is delivered to all nodes.
* *Totally ordered delivery*: messages are delivered to every node in the same order.

**Using total order broadcast**

Consensus services like ZooKeeper implement total order broadcast.

It is also needed for db replication, known as *state machine replication*.

Total order broadcast is a way of creating a *log*; a node cannot insert a message retroactively into an earlier position.

Total order broadcast is useful for implementing a lock service that provides fencing tokens.

**Implementing linearizable storage using total order broadcast**

Total order broadcast is asynchronous: messages are guaranteed to be delivered reliably in a fixed order, but there is no guarantee about *when* a message will be delivered. By contrast, linearizability is a recency guarantee: reads are guaranteed to see the latest value.

You can build linearizable storage on top of total order broadcast though.

**Implementing total order broadcast using linearizable storage**

A linearizable compare-and-set register and total order broadcast are *equivalent* to *consensus*.

### Distributed Transactions and Consensus

Consensus is *getting several nodes to agree on something*. Situations where it is important:

* *Leader election*
* *Atomic commit*: ACID transaction atomicity across several nodes - all nodes have to agree on the outcome of the transaction (either all commit or abort).

The FLP result proves that achieving consensus is impossible, but it was proved on a restrictive system model. In practice, with timeouts, achieving consensus is possible.

#### Atomic Commit and Two-Phase Commit (2PC)

**From single-node to distributed atomic commit**

On a single node, atomicity is implemented by the storage engine when it writes the commit record to disk (even if the node crashes, if the commit record was written to disk, the transaction was successful).

What if there are multiple nodes (e.g. multi-object transaction in a partitioned db or term-partitioned secondary index)? It is not sufficient to allow each node to commit independently. If the commit succeeded on some nodes and failed on others, it would violate atomicity.

If some nodes commit the transaction but others abort it, the nodes become inconsistent.

**Introduction to two-phase commit**

2PC is an algorithm for achieving atomic transaction commit across multiple nodes. 2PC uses a *coordinator* (aka *transaction manager*).

A 2PC transaction begins with the application reading/writing on multiple nodes as normal. Each node is a *participant* in the transaction. When the application is ready to commit, the coordinator begins phase 1: sending a *prepare* request to each node:

* If every participant replies "yes", the coordinator sends a *commit* request in phase 2.
* If any participant replies "no", the coordinator sends an *abort* request to all nodes in phase 2.

**A system of promises**

1. When a participant receives a prepare request it makes sure that it can definitely commit the transaction under all circumstances. By replying "yes", the node promises to commit the transaction without error if requested and surrenders the right to abort, but without actually committing yet.
1. When the coordinator has received responses to all prepare requests, it makes a definitive decision on whether to commit or abort (committing only if it got all "yes" votes). At this *commit point*, it writes that decision to a transaction log on disk (in case it crashes).
1. The decision is then sent to all participants and the requests must be retried indefinitely until they succeed. If a participant has crashed, it will commit when it recovers.

There are two "points of no return": when a participant votes "yes" and once the coordinator decides. These ensure the atomicity of 2PC.

**Coordinator failure**

Once a participant has received a prepare request and voted "yes", it must wait to hear back from the coordinator. If the coordinator crashes or the network fails, the participant can do nothing but wait. It's transaction is called *in doubt* or *uncertain*.

#### Distributed Transactions in Practice

Distributed transactions, especially those implemented with 2PC have a heavy performance penalty.

There are two different types of distributed transactions:

* *Database-internal distributed transactions*: all nodes participating are running the same db software.
* *Heterogenous distributed transactions*: participants are two or more different technologies.

**Exactly-once message processing**

Heterogenous: A message from a message queue can be acknowledged as processed if and only if the db transaction for processing the message was successfully committed.

**XA transactions**

X/Open XA is a standard for implementing 2PC across heterogenous technologies.

**Holding locks while in doubt**

The db cannot release locks until the transaction commits or aborts which means in a 2PC, a transaction must hold onto locks throughout the time it is doubt.

**Recovering from coordinator failure**

In practice, orphaned in-doubt transactions do occur (e.g. a transaction log is lost or corrupted). In that case, an administrator has to manually decide to commit or abort. Or a participant can use a *heuristic* (i.e. *break atomicity*) decision.

**Limitations of distributed transactions**

* If the coordinator runs on one machine, it is a SPOF.
* XA cannot implement SSI across different systems.
* In 2PC, *all* participants must respond, so if *any* part of the system is broken, the transaction fails. This *amplifies failures*.

#### Fault-Tolerant Consensus

In the consensus problem: one or more nodes may *propose* values and the consensus algorithm *decides* on one of those values. E.g. several people booking a single seat on a plane/in a theater, or registering a username.

A consensus algorithm must satisfy the following properties:

* *Uniform agreement*: no two nodes decide differently.
* *Integrity*: no node decides twice.
* *Validity*: if a node decides the value *v*, then *v* was proposed by some node.
    * Rules out trivial solutions.
* *Termination*: every node that does not crash eventually decides some value.
    * A liveness property. Formalizes fault tolerance: algo must progress even if a node fails.
    * Subject to fewer than half the nodes are crashed/unreachable.

**Consensus algorithms and total order broadcast**

The most common consensus algorithms are Viewstamped Replication, Paxos, Raft, and Zab. Most of these decide on a *sequence* of values, which makes them *total order broadcast* algorithms. It's equivalent to performing multiple rounds of consensus.

**Epoch numbering and quorums**

The consensus protocols define an *epoch number* (aka *ballot*, *term*, or *view number*) and guarantee within each epoch, the leader is unique.

Every time the current leader is thought to be dead, a vote is started among the nodes to elect a new leader. This election is given an incremented, monotonically increasing epoch number. If there is a conflict, the leader with the higher epoch number prevails.

Before a leader decides anything, it must check there isn't another leader with a higher epoch number. It must collect votes from a *quorum* of nodes. For every decision, it must send the proposal to other nodes and wait for a quorum to respond in favor. A node votes in favor if it is not aware of any other leader with a higher epoch.

Two rounds of voting: once to chose a leader, a second to vote on a leader's proposal. The quorums for these two votes must overlap: if a vote on a proposal succeeds, at least one of the nodes that voted for it must have also participated in the leader election. Thus, the leader knows it is still the leader if no other higher-number epoch is revealed.

**Limitations of consensusa**

* The process by which nodes vote on proposals is a kind of synchronous replication, which has performance implications.
* Consensus systems require a strict majority of nodes to operate, which is bad in the case of network failures.
* In the case of highly variable network delays and because of the use of timeouts, frequent leader elections result in poor performance.

#### Membership and Coordination Systems

ZooKeeper is modeled after Google's Chubby lock service. Usually don't use directly, but as part of another project, e.g. HBase, Hadoop, Kafka.

Data is replicated across all nodes using a fault-tolerant total order broadcast algorithm. Also useful:

* *Linearizable atomic operations*: you can implement a lock so that if several nodes try to concurrently perform the same operation, only one will succeed.
    * A distributed lock is usually implemented as a *lease*, which has an expiry time in case of client failure.
* *Total ordering of operations*: allows the use of a *fencing token* to prevent client conflicts in the case of process pause.
* *Failure detection* Client and ZooKeeper servers exchange heartbeats so it knows when a node is dead and can automatically release locks held by that session (*ephemeral nodes*).
* *Change notifications* Clients can watch for changes of other clients without polling.

**Allocating work to nodes**

ZooKeeper/Chubby model works well if you have several instances of a process/service and need to choose a leader, choose a node for a job scheduler, or decide which partition to assign to which node when reblancing. If done correctly, these types of tasks can be done automatically without human intervention.

ZooKeeper typically runs on a fixed number of nodes (3 or 5) and supports many clients. The kind of data managed by ZooKeeper is slow changing (e.g. "the node running on 10.1.1.23 is the leader for partition 7") which changes on the order of minutes or hours.

**Service discovery**

ZooKeeper is often used for *service discovery*: to find which IP address to reach which service. When a service starts up, it registers itself so it can be found. Unclear if this actually needs consensus, since DNS works fine.

**Membership services**

It is often useful for the system to agree which nodes are considered alive and which are not.
