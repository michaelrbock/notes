# Designing Data-Intensive Applications - Part II

[Goodreads](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications)

-

### Why do we need to distribute a database?

* *Scalability*
* *Fault tolerance/high availability*
* *Latency*

### Scaling to Higher Load

**Vertical scaling**

* *Shared-memory architecture*: increase CPU, RAM, disk and treat as single machine.
* *Shared-disk architecture*: several machines with shared disk (e.g. *NAS*); limited by locking overhead.

#### Shared-Nothing Architectures

* Also called *horizontal scaling*.
* Each machine is called a *node* and has it's own CPU, RAM, Disk. Coordination happens at software layer.
* No special hardware required. Can distribute across geographic regions for lower latency and reliability.
* Downsides: additional complexity and requires caution from application developer.

#### Replication Versus Partitioning

* *Replication*: keeping a copy of the data on several different nodes.
* *Partitioning*/*Sharding*: Splitting a big database into smaller subsets with different partitions on different nodes.
* Both can be used in concert (e.g. 2 replications of 2 partitions).

## Chapter 5 - Replication

Reasons for *Replication* (keeping a copy of same data on multiple machines):

* To keep data geographically close to users (reduce latency).
* Increase availability.
* Scale out number of machines serving reads (increase read throughput).

In this chaper, assume dataset is small enough that each machine can hold an entire copy (no sharding needed). The difficulty in replication lies in handling *changes* to the data over time. Three popular algos for replicating changes between nodes: *single-leader*, *multi-leader*, *leaderless*.

### Leaders and Followers

* Each node that stores a copy of the db is called a *replica*.
* How to ensure each replica has the same copy of the data? Each write needs to be processed by every replica.
* Most common solution is *leader-based replication* (aka *maser-slave*), which works by:

1. One replica is designated the *leader* (aka *master*, *primary*). Writes must be sent to the leader and are written to its local storage.
1. Other replicas are *followers* (aka *slaves*, *read replicas*). When the leader writes new data, it also sends the change to all of its followers as part of *replication log* (aka *change stream*). Each follower takes teh log and applies the updates in the same order.
1. Reads can be handled from any replica.

#### Synchronous vs. Async Replication

* The master can choose to wait for the follower to confirm the write before confirming the entire write as successful (*synchronous*).
* Or, the leader sends the write to a follower, but then doesn't wait for it's response (*asynchronous*).
* Usually replication is fast (<1 sec), but sometimes (if a follower is recovering from a failure, the system is near capacity, or there are network problems) then replication could be delayed.
* The advantage of synchronous is that the follower has the most up-to-date copy of the data, but the problen is that if the synchronous follower doesn't respond, the write cannot be processed and leader must block until synchronous replica is available.
    * Therefore, not all followers can be synchronous. In practice, this usually means that just *one* follower is synchronous and the rest are async (called *semi-synchronous*).
* Often, leader-based replication is completely async, so if the leader fails unrecoverably, any unreplicated writes are lost, but the leader can process writes even if followers are behind.
    * Even with weakened durability, this scheme is widely used.

#### Setting Up New Followers

Setting up a follower without downtime while the system is in flux:

1. Take a consistent snapshot of the leader's db at a point in time (without locking entire db).
1. Copy snapshot to new follower node.
1. Follower connects to leader and requests all changes since snapshot (therefore snapshot has to be associated with a position in the replication log).
1. When the follower has processed the backlog, it is *caught up* and can now continue to process live changes.

#### Handling Node Outages

Any node can go down (planned or unplanned) without bringing down the whole system.

**Follower failure: Catch-up recovery**

* Each follower keeps on local disk a log of the changes it has received from the leader.
* If it crashes or network is interrupted, it can request all of the data changes since the disconnect from the leader.

**Leader failure: Failover**

In *failover*, one of the followers needs to be promoted to be the new leader, clients needs to be reconfigured to send their writes to the new leader, and other followers need to start consuming data changes from the new leader. Steps:

1. *Determine the leader has failed*: usually by checking if it has timed out.
1. *Choose a new leader*: election process or appointed via a *controller node*. Try to choose most-up-to-date. This is a consensus problem.
1. *Reconfigure the system to use the new leader*: Clients need to send write requests to new leader. The old leader (if it returns) must be forced to become a follower.

Things that can go wrong in failover:

* With async replication, the new leader may not have all of the writes from the old leader before it failed. If the former leader re-joins, what should happen to those writes? The new leader may have received conflicting writes in the meantime. Usually, the old leader's writes are discarded (violating durability).
* Discarding writes is dangeous if other systems outside db are coordinated.
* *Split brain*: two nodes both think they are the leader. Could lead to conflicting writes.
* What is the right timeout before a leader is declared dead?

### Implementation of Replication Logs

#### Statement-based replication

Simply log each SQL statement (e.g. `INSERT`, `UPDATE`, `DELETE`) in the log and each follower parses and executes the query. Problems:

* Any nondeterministic function (e.g. `NOW()` or `RAND()`) in the statement.
* Statements with an autoincrementing counter or depend on some existing data (e.g. `WHERE`) need to be executed in exactly the same order.
* Statements with side-effects if not deterministic.

Other replications strategies are now preferred.

#### Write-ahead log (WAL) shipping

Uses the append-only log containing all sequnces of writes to build exact same data structures as on leader.

Disadvantage is that you cannot use a different version of the db storage engine on leaders/followers, making rolling upgrades impossible.

**Logical (row-based) log replication**

To decouple the storage engine internals from replication, you can use an alternative log format, a *logical log* (vs. the *physical* one). A sequence of records containing:

* For inserted row, all new column values.
* For deleted row, unique id (primary key) to delete row.
* For updated row, unique id and new values.

Transactions are noted and may span several rows of the log. Can be backward compat and allows nodes to run different versions of db software. Easier for other applications, such as data warehouses or custom indexes/caches, to parse the log, called *change data capture*.

**Trigger-based replication**

Replication at the application layer (instead of db layer) if you want more flexibility (e.g. replicate subset of data or from one db system to another).

*Trigger*: register custom application code that is automatically executed when a data change (write transaction) occurs. (More overhead than built-in replication).

### Problems with Replication Lag

*Read-scaling architecture*: create many followers and distribute read requests across them (common in read-heavy web apps). Only async replication is feasible (in sync, one broken follower would bring down entire system from handling writes). But, async followers may have out-of-date information.

This inconsistency is a temporary state (if you stopped writing and waited, the followers would catch up) and is called *eventual consistency*. Usually this *replication lag* is small, but it can be higher near capacity or with network problems.

#### Reading Your Own Writes

If a user writes data, then immediately reads it (as is common), it would be bad if the async follower the read request went to didn't yet have the write. We need *read-after write consistency* (aka *read-your-writes consistency*): guarantees that a user will see any updates they submitted (not necessarily other users though). How to implement:

* Read something a user may have modified from the leader.
* If most things are user-editable, the above doesn't allow for read scaling benefits of replication. Instead, have other heurisitics for when to read from the leader (e.g. within a certain amount of time or track replication lag to followers).
* Client remembers timestamp of most recent write and ensures replica serving read is up-to-date or waits. Timestamp could be a *logical timestamp* (indicating order of writes) or actual system clock (needs clock synchronization).
* If spread across multiple DCs, additional complexity.

Additional complication if user accesses service from multiple devices, in which case you need *cross-device* read-after-write consistency:

* Need to centralize metadata of user's last write.
* Different devices may be routed differently along the network (e.g. home internet vs mobile broadband).

#### Monotonic Reads

Avoiding possibility of user seeing data *moving backwards in time*. This could happen if user first reads from a replica with little replication lag, but then reads from a replcia with a lot of replication lag. *Monotonic reads* guarantee this doesn't happen:

* Lesser guarantee than strong consistency, but stronger guarantee than eventual consistency.
* The user may see older data, but never read older data after previously reading newer data.
* Can be implemented by making all reads for a user go to the same replica (e.g. by hash of user ID).

#### Consistent Prefix Reads

Ensuring that if a sequence of writes happens in a certain order, anyone reading those writes sees them appear in the same order. i.e. an observer doesn't see the answer to a question before the question itself.

* A particular problem in sharded dbs.

#### Solutions to Replication Loag

* When working in an eventually consistent system, must think about what happens if replication lag increases to min/hours. Don't pretend replication is synch if it's asynch.
* Dealing with issues in app code is complex. This is why *transactions* exist: to handle at db layer and make apps simpler.

### Multi-Leader Replication

* Downside to leader-based replication is all writes must go thruough the one leader (or one leader per partition).
* If we have multiple nodes accept writes, with each node forwarding data changes to all other nodes, each node acts as a leader and a follower to other nodes. 
* This is called *multi-leader* configuration (aka *master-master* or *active/active* replication).

#### Use Cases for Multi-Leader Replication

It rarely makes sense to use a multi-leader setup within a single datacenter: the added complexity does not outweigh benefits.

**Multi-datacenter operation**

Have one leader in *each* datacenter, instead of one overall. Each leader replicates its changes to the leaders in other datacenters.

* *Performance*: writes don't all have to go to one datacenter. Every write is processed locally and replicated asychronously to other datacenters, thus inter-datacenter network delay is hidden from users.
* *Tolerance of datacenter outages*: each datacenter can operate independently of others even with a failed datacenter. Replication catches up when it comes back online.
* *Tolerance of network problems*: inter-datacenter traffic usually goes over the public internet. Multi-leader can handle network problems.

Problem with multi-leader is handling conflicting writes to two different datacenters. There are also many subtle configuration pitfalls and interactions with other db features.

**Clients with offline operation**

E.g. a calendar app across multiple devices. Each device has a local db acting as a leader that can accept writes. Sync (replication) happens async when network is available.

**Collaborative editing**

*Real-time collaborative editing*, e.g. Google Docs. Changes are instantly applied to user's local replica and async replicated to server and other users. By making unit of change small (e.g. single keystroke) you can avoid locking but bring the challenges of conflict resolution.

#### Handling Write Conflicts

* Write conflicts can be caused by two leaders concurrently updating the same record.
* Each write is applied to local leader and conflict is only detected async when replicated.

**Sync vs. async conflict detection**

If you want synchronous conflict detection, just use single-leader replication.

**Conflict avoidance**

Application makes sure all writes for a certain record goes through the same leader. From the user's perspective, the system is single-leader. Breaks down if we need to change designated leader for a record, e.g. because of datacenter failure.

**Converging toward a consistent state**

Each replica must arrive at the same final value when all changes have been replicated: conflicts must be resolved in a *convergent* way. Ways to do it:

* Give each write a unique ID (e.g. timestamp, UUID, hash of key/value), pick write with highest ID as the *winner*. If a timestamp is used, this is called *last write wins* (LWW), which is dangerously prone to data loss.
* Give each replica a unique ID and the highest-ID replica wins. This also implies data loss.
* Somehow merge the values together.
* Record the conflict and have application code deal with it at some point later, e.g. by prompting the user.

**Custom conflict resolution logic**

Most multi-leader systems allow you to write custom application logic to deal with conflicts, which is executed either:

* *On write*: as soon as the db detects a conflict, it calls the conflict handler.
* *On read*: all conflicting writes are stored and next time data is read, multiple versions are returned so user/app can resolve.

Conflict resolution usually applies at individual row/document level, not for an entire transaction.

### Multi-Leader Replication Topologies

*Replication topology* describes the communication paths along which writes are propogated. Types:

* Most common: *all-to-all*, in which every leader sends its writes to every other leader.
    * Fault tolerance is easier in connected topologies, there is no SPOF.
    * Possible issue of causality, where writes arrive in the wrong order on some replicas. Just using a timestamp is not enough. Can use *version vectors*.
* *Circular*: writes may need to pass thru many nodes before it reaches all replicas.
* *Star*: one designated node forwards writes to all other nodes.

### Leaderless Replication

Aka *Dynamo style*. Any replica can accept writes.

#### Writing to the Database When a Node is Down

Writes got to multiple nodes and are considered successful if at least *n* writes are successful. But if a node was down during write, when it comes back online, if you read from that node it may have *stale* data. So *read requests are sent to several nodes in parallel*. Version numbers are used to determine which value is newer.

**Read repair and anti-entropy**

After an unavailable node comes back online, how does it catch up on writes it missed? Either:

* *Read repair*: when a client makes a read from several nodes, it can detect if one node has a stale value and write the newer back to that replica. Works for values that are frequently read.
* *Anti-entropy process*: background process to look for differences.

**Quorums for reading and writing**

If there are *n* replicas, every write must be confirmed by *w* nodes to be considered successful, and we must query at least *r* nodes for each read. And this must hold: *w + r > n*, called *quorum* reads/writes.

These values are usually configurable. A typical set up is to make *n* an odd number (typically 3 or 5) and *w = r = (n + 1) / 2 rounded up*. This allows us to tolerate some unavailable nodes (as long as the required *w* and *r* nodes are available).

#### Limitations of Quorum Consistency

Usually, *r* and *w* are chosen to be a majority of the nodes because that allows uo to *n/2* node failures. You can also set *w* and *r* to smaller numbers so that *w + r <= n* (doesn't satisfy quorum condition). You are more likely to read stale values, but also allows lower latency and higher availability. Still, there are edge cases:

* Sloppy Quorum.
* If two writes happen concurrently, it is not clear which happened first, must merge writes. If using LWW, writes can be lost due to clock skew.
* Writes and reads could happen concurrently, and it's undetermined whether the read returns the old or new value.
* If a write succeeds on some nodes but fails on others it is not rolled back on the replicas.
* If a node carrying a new value fails, and the data is restored for a node carrying an old value.
* Other timing issues.

Dynamo-style dbs are usually for use cases where eventual consistency is OK. You do not get to use read your writes, monotonic reads, or consistent prefix reads.

**Monitoring staleness**

Easy to measure in leader-based replication, but hard to measure in leaderless. Would be good to know how long "eventual" is.

#### Sloppy Quorum and Hinted Handoff

Dbs with appropriately conffigured quorums can tolerate individual node problems and are good for high availability and low latency that can tolerate stale reads. However, if cut off from many nodes, hard to reach *w* or *r*. Trade-off:

* Better to return an error when we cannot reach quorum of *w* or *r* nodes?
* Or should we accept writes anyway and write them to some reachable nodes, but not the *n* nodes on which the value usually lives (aka *sloppy quorum*).
    * Writes and reads still require *w* and *r* successful responses, but those may include nodes that are not among the designated *n* "home" nodes for a value.
    * When the home node returns online, the temporarily written data is sent to the correct node (aka *hinted handoff*).
    * Increases write availability, but can't be sure you have the latest value.
    * Assurance of durability, not a real quorum.

#### Detecting Concurrent Writes

Users can write values for keys concurrently, and network delays or partial failures can get db into an inconsistent state. The application dev must know about this!

**Last write wins (discarding concurrent writes)**

Attach a timestamp to each write and pick the most "recent" as the winner. LWW achieves convergance at the cost of durability. This may be OK for caching, but not when data loss in unacceptable. Also has timing issues. OK if you treat each key as immutable after written.

**The "happens before" relationship and concurrency**

Two operations: A and B.

* A can *happen before* B if B knows about A, relies on A, or builds on A.
* B can happen before A. 
* Or A and B can happen concurrently.

**Capturing the "happens before" relationship**

Example of two clients both adding items to a shared shopping cart. Each write of the key is assigned a version number which allows the server to know when a concurrent write happens (clients send up theWhelast version number of the key they've seen). Algorithm:

* The server maintains a version number for every key, increments it when the key is written and stores version number and value.
* When a client reads, the server returns all non-overwritten values and the last version number. A client must read (response from write can be a read) before writting.
* When a client writes a key, it must include the version number from the prior read *and* merge together all values it received from the prior read.
* When the server receives a writes with a particular version number, it can overwrite all values with that version number or below (since it knows they have been merged), but it must keep all values with a higher version number (bc those writes are concurrent).
* If you make a write without a version number, simply add that value without overwritting.

**Merging concurrently written values**

The client must intelligently merge *sibling* (concurrently-written) values. You could use LWW, but that implies data loss. Or do something more intelligent, like taking the union of values. If you do that, in order to *remove* items from the set, must use a *tombstone* marker or deleted items will keep being re-added.

#### Version vectors

With multiple replcas, we need a version number *per-replica* and *per-key*. Each replica will increment its own version number on writes and keep track of which version numbrs it has seen from other replicas. This is called a *version vector*. It allows writes to go to different replicas without data loss.

## Chapter 6 - Partitioning

For very large datasets or very high query throughput, replication is not enough: we need to break the data up into *partitions* (also kown as *sharding*).

Usually, data belongs to exactly one partition. Each partition is a db of its own, though we may support operations that touch multiple partitions at the same time. The reason for partitioning is *scalability*: different shards on different nodes, which scales data size and query throughput. Queries on single partitions can independently execute and complex queries can be done in parallel.

### Partitioning and Replication

Partitioning is usually paired with replication (copies of each partition are stored on multiple nodes). Each node can act as a leader for some partitions and a follower for other partiions because a node may store more than one partition.

### Partitioning of Key-Value Data

How to decide which records to store on which nodes? Our goal with partitioning is to spread the data and query load evenly across *n* partitions so *n* nodes can handle *n* times the read/write throughput of a single node.

If the partitioning is *skewed*, some partitions have more data than other (all on one node in the extreme case). This node becomes a bottleneck, aka a *hot spot*.

#### Partitioning by Key Range

* Partition data by assigning contiguous range of keys to each partition, like an encyclopedia.
* Each partition is not evenly spaced (A-B vs W-Z, e.g.).
* Can keep records sorted on each partition for easy range scans.
* Could lead to skew/hotspots, which can be mitigated by adding additonal info to key.

#### Partitioning by Hash of Key

* Because of skew/hotspots, take hash value of key, and assign ranges of hash value to partitions.
    * Hash function takes skewed data and makes it uniformly distributed.
* Lose the ability to do fast range queries (range scans have to be sent to all partitions).
* Can compromise by having a *compound primary key* so that you can do efficient range scans on the other columns of a key for a given first column.

#### Skewed Workloads and Relieving Hot Spots

* Even with hashing, hotspots can occur (e.g. a celebrity's user_id or id of their post).
    * Application's job to relieve these types of hotspots, e.g. by adding number to the front of hot keys.
        * Then reads will have to read from all these keys as well.

### Partitioning and Secondary Indexes

#### Partitioning Secondary Indexes by Document

* Each partition is completely separate: contains its own secondary index pertaining only to the documents on that partition, a *local index*.
* But for reads, you will have to read data from *all* partitions: known as *scatter/gather* and is expensive because even though done in parallel, is prone to tail latency amplification.

#### Partitioning Secondary Indexes by Term

* Instead of having a *local index*, we can have a *global index* that covers data from all partitions.
    * But we can't have that index on just one node or that node will become a bottleneck, so:
    * A global index must be partitioned as well, but can be differently from the primary key index.
    * E.g. we can partition based on the terms (or hashes of terms) in the index, called *term-partitioned*.
        * Partition by term gives better range scans, partition by hash gives more even distribution of load.
    * Writes are more complicated, but reads are faster.
    * In practice, writes to these indexes are async.

### Rebalancing Partitions

*Rebalancing*: moving data/requests from one node to another, because of:

* Need for additional CPU for throughput,
* Dataset size increase calls for additional disk/RAM,
* Machine fails and need other machines to take over.

Rebalancing must:

* Share load evenly across nodes,
* Keep database accepting reads/writes during rebalancing,
* Move the minimum amount of data necessary.

#### Strategies for Rebalancing

How *not* to do it: **hash mod N**. Would have to move *all* data around when N changes.

**Fixed number of partitions**: create many more partitions than there are nodes (e.g. 1000 partitions for 10 nodes) and when a node is added to the cluster, it can *steal* a few partitions from every existing node until evenly distributed again. Partition sizes are usually fixed and need to choose just the right number.

**Dynamic partitioning**: when a partition grows beyond some size, it can be split into two equal-sized partitions. Similarly, when it shrinks below some size, a partition can be merged into an adjacent one.

**Partitioning proportionally to nodes**: number of partitions is proportional to the number of nodes (instead of size of data). A new node selects some partitions from each existing node randomly to split and take over half of. Must be using hash-based partitioning.

#### Operations: Automatic or Manual Rebalancing

* Manual: admin explicitly assigns partitions to nodes.
* Fully automatic: system decides automatically.
    * Must be careful because rebalancing can be expensive. Could lead to cascading failures.
* Partial: system generates suggestions which are committed by admin.

### Request Routing

How does a client know which node to connect to? Specific case of more general problem known as *service discovery*. Approaches:

1. Allow clients to connect to any node, if that node can handle request, great, otherwise, it forwards request to appropriate node, recieves the reply, and passes reply back to client.
1. Send all requests first to a routing tier first which determines which node should handle each request.
1. Require clients be aware of partitioning and connect directly to appropriate node.

Hard problem because all participants must agree (consensus in distributed system). Many systems rely on a separate coordination service like ZooKeeper which maintains authoritative mapping:

* Nodes register themselves in ZooKeeper.
* Routing tier or partition-aware client subscribes to this information.

### Parallel Query Execution

*Massively parallel processing* (MPP) relational dbs, used for analytics, support complex queries and break up query into stages which can be executed in parallel across machines.
