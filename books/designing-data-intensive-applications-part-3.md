# Designing Data-Intensive Applications - Part III - Derived Data

[Goodreads](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications)

-

In a realistic large application, there will be a combination of several different datastores and we'll need to integrate them into one coherent application architecture.

### Systems of Record and Dervied Data

There are two categories of data systems:

* *Systems of record*: aka the *source of truth*. When new data comes in, this is where it goes. The data is usually *normalized*.
* *Derived data systems*: the result of taking some existing data from another system and transforming/processing it. Classic examples are a cache, denormalized values, indexes, and materialized views.
    * It is *redundant*, commonly *denormalized*, but often crucial for getting good read performance.

## Chapter 10: Batch Processing

Types of systems:

* *Services (online systems)*
    * Waits for requests, handles them ASAP and responds. Response time & availability are primary metrics.
* *Batch processing systems (offline systems)*
    * Takes a large amount of input data, runs a *job*, and produces some output data. Primary metric is *throughput*.
* *Stream processing systems (near-real-time systems)*
    * Between online and batch processing. Like batch, stream consumes input and produces outputs (as opposed to respoding to requests), but does so shortly after events happen.

### Batch Processing with Unix Tools

An example of finding the five most popular pages on your website via logs using Unix tools or Ruby.

**Sorting versus in-memory aggregation**

If the *working set* of your job is bigger than can fit in memory, you must make use of disks. The `sort` command in Unix automatically handles larger-than-memory datasets by spilling to disk and automatically parallelizes sorting across cores.

#### The Unix Philosophy

The *Unix philosophy*:

* Make each program do one thing well.
* Expect the output of one program to become the input to another program.
* Design and build software to be tried early.
* Use tools to lighten a programming task (automation).

**A uniform interface**

In Unix, the uniform interface is a file (file descriptor): an ordered sequence of bytes. It can represent a file on the filesystem, a socket, `stdin`, `stdout`, a device driver, a TCP connection, etc.

Unix tools interoperate really well because of this.

**Separation of logic and wiring**

Programs in Unix don't care about where input is coming from or where the output is going (it's just `stdin` and `stdout`): *loose coupling*, *late binding*, or *inversion of control*. Separating the input/output wiring from the program logic makes it easier to compose small tools into bigger systems.

You can even write your own tools and combine them with OS-provided tools as long as you use `stdin`/`stdout`.

**Transparency and experimentation**

Unix tools make it easy to debug:

* Inputs are treated as immutable.
* You inspect the output at any point (just pipe to `less`).
* You can write the output of one stage of a pipeline to a file and restart with that as input to the next stage.

### MapReduce and Distributed Filesystems

MapReduce is like Unix tools but distributed across lots of machines: takes input(s) and produces output(s) without side effects.

MapReduce jobs read and write files on a distributed filesystem. In Hadoop, this is HDFS, and open-source version of Google File System.

HDFS is based on *shared-nothing*, which requires no special hardware. HDFS has a daemon process running on each machine exposing a network service that allows other nodes to access files stored on that machine. A central server called the *NameNode* keeps track of which file blocks are stored on which machine. Thus, HDFS is conceptually like one big filesystem that uses the disk space of every machine in the network.

In order to tolerate failures, file blocks are replicated on multiple machines: either via storing several copies or using an *erausre coding* scheme.

#### MapReduce Job Execution

MapReduce is a framework for writing code to process large datasets in a distributed filesystem.

To create a MapReduce job, you implement two callback functions:

* *Mapper*
    * Called once for every input record and extracts any number (0+) of (key, value) pairs. Keeps no state.
* *Reducer*
    * The framework collects all values with the same key and calls the reducer with an iterator over that collection of values. The reducer can produce output records.

The role of the mapper is to put data into form suitable for sorting, and the role of the reducer is to process the data that has been sorted.

**Distributed execution of MapReduce**

MapReduce can parallelize a computation across many machines without you having to write code to explicitly handle parallelism. Because the mapper and reducer operator on one record at a time, they don't need to know where input came from or where output is going - the framework handles this.

Parallelization is based on partitioning: the input is a directory and each file/block is a partition. Because each file is large, the scheduler tries to run each mapper on one of the machines that stores a replica of that file: *putting the computation near the data*, saving network load and copying.

The framework copies the code to the appropriate machines and starts the map task.

The framework uses a hash of the key to determine which reduce task should receive a particular key-value pair. Each map task partitions it output by reducer.

The scheduler notifies the reducers when they can start fetching the output files of sorted key-value pairs for their partition from the mapper. This process is called *shuffling*. The reducer takes files from different mappers and merges them together, maintaining sort order. Then the reducer processes these records and produces output.

**MapReduce workflows**

Each MapReduce job is fairly limited, e.g. can determine the number of page views per URL, but not the most popular URLs because that requires a second round of sorting. So it's very common to chain MapReduce jobs together into *workflows*: the output of one job becomes the input to the next job.

Hadoop doesn't have support for this, but there are workflow schedulers to handle this like Airflow and higher-level tools like Pig and Hive.

#### Reduce-Side Joins and Grouping

Joins in a batch process are different than normal joins in a database, which often use an index to quickly find the records. There are no concepts of an index in MapReduce. *Full table scans* are common in MapReduce jobs, so we want to resolve all occurrences of some association, not just one.

**Example: analysis of user activity events**

Let's say we have a log of activity events (with a user ID attribute) and a user database. An analytics task may want to correlate user activity with some user profile information, so the activity events need to be joined with the user db.

We can't just query the user db for every user ID, the performance would be bad, limited by the round-trip time to the db server. A better approach is to take a copy of the user db via ETL and put it in the same distributed filesystem.

**Sort-merge joins**

One set of mappers goes over the activity events and output the key as the user ID and another set of mappers does the same with the user db. Then the framework sorts by user ID and all user records become adjacent in the reducer. MapReduce could arrange the record such that you see the user database record first then the activity events in timestamp order on the reducer. This is known as *secondary sort*.

The reducer can then perform the actual join easily because it is called once for each user ID.

**Brining related data together in the same place**

Mappers "send messages" to the reducers. When a mapper emits a key-value pair, the key acts like the destination address to where the value should be delivered. Even though the key is just an abitrary string, all key-value pairs with the same key will be delivered to the same reducer.

MapReduce therefore separates the physical network communication (getting data to the right machine) from the application logic (processing the data once you have it). This contrasts from the normal use of a database where the requst for data happens deep in some application code. MapReduce shields application code from having to deal with partial failures.

**GROUP BY**

Grouping records by some key is another common pattern, often to perform some aggregation, e.g. counting the number of records, summing values.

This is implemented very similarly to joins in MapReduce: by having the mappers emit the key-value pairs with the desired grouping key.

One common use for grouping is collating all the activity events for a user session, called *sessionization*, if a users activity events are scattered across various servers' log files.

**Handling skew**

The pattern of "brining all records with the same key to the same place" breaks down if there is a very large amount of data related to a single key, for example a celebrity's data on a social network. Such disproportionately active db records are known as *linchpin objects* or *hot keys*.

Collecting all data related to a hot key in a single reducer leads to significant *skew* (aka *hot spots*): a reducer that must process significantly more records than the others.

The *skewed join* or *sharded join* methods are used to compensate for *skew*. After running a sampling job to test for skew, skewed join in Pig sends record from a hot key to one of several reducers at random. The other input to the join has to be replicated to *all* reducers handling that key. *Sharded join* requires hot keys be specified explicitly.

Or you could perform the grouping in two stages: the first sends records to a random reducer, so each reducer performs the grouping on a subset of records for the hot keys and outputs a more compact aggregate value per key. The second job then combines the values from the first stage.

#### Map-Side Joins

In a *reduce-side join*, you don't need to make any assumptions about the input data, but it comes at a performance cost because of sorting, copying to reducers, and merging.

If you *can* make assumptions about the input, it can make joins faster by using a *map-side join* in which there are no reducers and no sorting.

**Broadcast hash joins**

If a large dataset is joined with a small dataset (that is small enough to fit in memory), then we can simply load the small dataset into memory on each mapper. This is known as a *broadcast hash join*: *broadcast* because the small input is "broadcast" to each partition of the large input and *hash* beaucase you can load the small data into a hash table.

**Partitioned hash joins**

If the inputs to a map-side join are partitioned in the same way, then the hash join approach can be applied to each partition independently. E.g. if each dataset is partitioned in the same way so all the relevant records are on the same partition.

**Map-side merge joins and MapReduce workflows**

If the input datasets are partitioned in the same way *and sorted* based on the same key. Probably because some prior MapReduce job put them in that state.

Knowing about the physical layout of datasets in the distributed filesystem is important for optimizing join strategies.

#### The Output of Batch Workflows

Batch processing fits somewhere between OLTP and analytics: the output is not just a report.

**Building search indexes**

The original use of MapReduce was to build search indexes. A simple search index is a term dictionary of term: [list of document IDs]. Mappers partition the set of documents as needed and the reducers build the index for its partition.

**Key-value stores as batch process output**

Another common use case for batch processing is building machine learning systems such as classifiers (spam filtering, image recognition) or recommendation systems. The output of those batch jobs is often some kind of database, e.g. user ID to suggested friends for that user. And these databases need to be queried from the application.

It is not a good idea to write directly to your database from inside a MapReduce job:

* Making a network request to the db is slower than the typical throughput of a batch task.
* MR runs in parallel, so could overwhelm the db.
* MR provides a clean all-or-nothing guarantee for the final output a job. If you produce side effects during the job then you have to worry about partially-completed jobs.

Instead, you should build a brand-new db *inside* the batch job which can then be bulk loaded into servers that handle read-only queries. Voldemort is an example: can server requests from old data while new files are copied over and then switch over.

**Philosophy of batch process outputs**

Like Unix philosophy, in batch processing it's important for performance and maintainability that: inputs are treated as immutable and you avoid side effects.

* If you introduce a bug in the code that produces wrong or corrupted output, you can simply roll back the code and rerun the job or just keep the old output.
* By *minimizing irreversibility* you can develop features faster because mistakes are not damaging.
* MR provides automatic retries for map or reduce tasks for transient failures which are safe because of immutable inputs.
* Separation of concerns between wiring and logic.

Unlike Unix though, Hadoop often uses structured file formats like Avro or Parquet.

#### Comparing Hadoop to Distributed Databases

Hadoop is somewhat like a distributed version of Unix, where HDFS is the filesystem and MapReduce is a quirky implementation of a Unix process (which happens to always run the `sort` utility between the map phase and the reduce phase).

MPP (*massively parallel processing*) databases focus on parallel execution of analytic SQL queries on a cluster of machines.

**Diversity of storage**

Databases require you to structure/model your data according to a particular model whereas files in HDFS can be anything. Hadoop allows you to indiscriminately dump data into HDFS and figure out how to process it later.

Whereas databases require careful modeling of your data upfront. In practice, simply making data available quickly, even if raw format, is more valuable than trying to decide on the ideal data model upfront. The burden of interpreting the data shifts from the producer to the consumer (schema-on-read). Different teams may have different priorities or ideal data models anyway. The *sushi principle*: "raw data is better". Hadoop is often used for ETL processes.

**Diversity of processing models**

MPP dbs are good for analytic SQL queries. But not all kinds of processing can be sensibly expressed as SQL queries. E.g. building machine learning systems, search indexes, or image analysis requires writing code, not just queries. MapReduce allows this.

Hive project shows that you *can* build a SQL query execution engine on top of HDFS and MapReduce. All these workflows can be run on the same shared-use cluster of machines, all accessing the same files on the distributed filesystem. E.g. HBase is an OLTP db on top of HDFS.

**Designing for frequent faults**

If a node crashes while a query is executing, an MPP database will retry the entire thing, which is OK because each query only takes seconds or minutes. MPP dbs also keep data in memory if possible.

MapReduce OTOH retries at the granularity of an individual map or reduce task and is eager to write to disk: for fault tolerance and because the data is too big to fit in memory usually anyways. MapReduce is more appropriate for larger jobs that expect at least one task failure along the way.

At Google, MR jobs run on shared resources, and MR tasks can be preempted to free up resources for higher-priority jobs. So task-level retries are necessary for better resource utilization. This isn't supported much in the Open Source world.

### Beyond MapReduce

There are higher-level abstractions created on top of MapReduce to ease developemnt: Pig, Hive, etc. But there are also some kinds of processing which have better performance with other tools. Stream processing is another alternative model to batch processing.

#### Materialization of Intermediate State

Every MapReduce job is independent from every other job. The only contact points of a job with the rest of the world are its input and output directories on the distributed filesystem. This is good if the output is something you want to publish: e.g. it is the input to several other jobs.

In many cases though, the ouput of one job is only used as the input to one other job: the files are simply *intermediate state*. The process of writing files of intermediate state is called *materialization*. By contrast, Unix pipes *stream* the output to input.

MapReduce materializing intermediate state has downsides:

* A MR job can only start when all tasks in the preceding jobs have completed. This can slow down the workflow as a whole if there are straggler tasks because of skew or varying load.
* Sometimes mappers are uncessary, what you really want is reducers chained together.
* Intermediate state stored in a distributed filesystem means files are replicated across several nodes.

**Dataflow engines**

*Dataflow engines*, like Spark, do not take strict roles of alternating map and reduce, but instead can be assembled in more flexible ways: each function is called an *operator*. They can be connected in multiple ways:

* Like in the shuffle stage of MR.
* Partitioned, but not sorted.
* By broadcasting like in broadcast hash joins.

Advantages of dataflow engines over MapReduce:

* Expensive work like sorting only happens where actually required rather than always happening by default between every map and reduce.
* No uncessary map tasks.
* Joins and data dependencies are explicitly declared, so the scheduler can make locality optimizations.
* Intermediate state can be kept in memory or written to local disk as opposed to HDFS.
* Operators can start as soon as input is ready.

Dataflow engines can implement the same computations as MapReduce workflows.

**Fault tolerance**

Without fully materializing intermediate state to HDFS, dataflow engines tolerate faults that lose intermediate state by recomputing it from other data that is still available (a prior intermediate stage or original input data).

To enable this, the framework must keep track of how a given piece of data was computed. When recomputing data it is important to know whether the computation is *deterministic*. If it's not, you must kill downstream operators as well. Sometimes, recomputing is not the right answer, e.g. if the intermediate data is much smaller than the source, it's better to materialize it.

#### Graphs and Iterative Processing

Dataflow engines typically arrange operators in a job as a directed acyclic graph (DAG): the *flow of data from one operator to another* is structured as a graph. Versus the *data itself* having the form of a graph as in graph processing.

The iterative nature ("repeat until done") of graph processing algorithms don't lend themselves to MapReduce. Instead, in the *Pregel* model, one vertex can "send a message" to another along the edges of the graph. In each iteration, a function is called for each vertex, along with all the messages that were sent to it (like a reducer). But unlike in MR, a vertex remember its state from one iteration to the next and only processes new incoming messages.

**Parallel execution** is hard in distributed graph algorithms because of cross-machine communication because its hard to partition vertices optimally.

#### High-Level APIs and Languages

Higher-level languages and APIs improve the process of writing a MapReduce job. They use relational-style building blocks. They can provide interactive development. And improve job execition efficiency.

## Chapter 11: Stream Processing

In a batch process, the input is bounded, but in many applications, data arrives over time and never ends. In order to batch process, you must break up the data arbitrarily by, say, day.

In *stream processing*, you process every event as it happens.

### Transmitting Event Streams

In batch processing, the inputs and outputs of jobs are files. In streaming:

* A record is known as an *event*, a small, self-contained, immutable object containing details of something that happened at some point in time (with a timestamp according to a time-of-day clock).
    * E.g. a user event or CPU measurement.
    * Stored as text, JSON, or binary form in some file/store/db.
* An event is generated by a *producer* (aka *publisher*, *sender*) and processed by multiple *consumers* (*subscribers*, *recipients*).
* Related events are grouped together by *topic* or *stream*.

Instead of polling a database, which isn't designed for this, it's better for consuemers to be notified when new events appear.

#### Messaging Systems

A common approach for notifying consumers about new events is to use a *messaging system*, which allows multiple producer nodes to send messages to the same topic and allows multiple consumer nodes to receive messages in a topic. This is called the *publish/subscribe* model.

To differentiate, you must ask two questions:

1. *What happens if the producers send messages faster than the consumers can process them?* Three options:
    * Drop messages, buffer messages in a queue, apply *backpressure* (aka *flow control*) by blocking the producer from sending more messages.
    * If buffering, what happens as that queue grows, e.g. can no longer fit in memory?
1. *What happens if nodes crash or temporarily go offline - are any messages lost?*
    * E.g. what is the durability?

Whether messages can be lost depends on the application: e.g. for sensor readings it may be OK to ocassionally lose a message, but if counting, it's not.

**Direct messaging from producers to consumers**

These require the application code to be aware of the possibility of message loss and for the producer and consumer to be 100% available. Examples:

* UDP multicast.
* Brokerless messaging libraries, e.g. ZeroMQ.
* StatsD
* Direct HTTP/RPC requsts, e.g. webhooks.

**Message brokers**

A *message broker* (or *message queue*) runs as a server/database that producers and consumers connect to. By centralizing, these systems can handle clients that connect, disconnect, or crash and durability is handled by the broker. Some message brokers only keep messages in memory, others write to disk so they are not lost in case of a broker crash.

Consumers are *asynchronous*: producers only wait for the broker to confirm, delivery to consumers will happen at some point in the future.

**Message brokers compared to databases**

* Message brokers usually delete a message after it has been successfully delivered to its consumers.
* Message brokers assume the queues are fairly short. Throughput may degrade if the broker needs to buffer many messages.
* Message brokers do not support arbitrary queries.

JMS and AMQP are message broker standards. Examples: RabbitMQ, ActiveMQ, ...

**Multiple consumers**

There are two patterns for multiple consumers reading messages in the same topic:

*Load Balancing*: each message is delivered to *one* consumer, so the consumers can share the work of processing the messages in the topic. Useful when messages are expensive to process.

*Fan-out*: each message is delivered to *all* consumers.

**Acknowledgements and redelivery**

Consumers may crash, so its possible a broker delivered a message but the consumer never/partially processed it. To ensure messages aren't lost, message brokers use *acknowledgements*: a client must explicitly tell the broker when it has finished processing a message so the broker can remove it.

If the connection to a client is closed or times out, the broker assumes the message wasn't processed and delivers the message to another consumer. Handling the case where a message *was* processed but the ack was lost by the network requires atomic commit.

Combining redelivery with load balancing means that consumers may see messages out of order. To avoid: don't use load balancing. This is only a problem if the messages are causally dependent.

#### Partitioned Logs

In batch systems, you can re-run jobs without damaging the input. In AMQP/JMS messaging: receiving a message is destructive if the ack causes it to be deleted from the broker. If you add a new consumer to the messaging system, it only starts receiving messages sent *after* it was registered.

The hybrid of durable storage approach of databases with low-latency notifications of messaging is the idea behind *log-based message brokers*.

**Using logs for message storage**

We can use a log (an append-only sequence of records on disk) to implement a message broker: a producer sends a message by appending it to the end of the log, and a consumer receives messages by reading the log sequentially. If a consumer reaches the end of the log, it waits for a notification that a new message has been appended, like `tail -f`.

In order to scale throughput, the log can be *partitioned*, and different partitions can be hosted on different machines. A topic is defined as a group of partitions that all carry messages of the same type.

Within each partition, the broker assigns a monotonically increasing sequence number, or *offset*, to every message. So messages *within* (not across multiple) a partition are totally ordered.

Apache Kafka, Amazon Kinesis are examples. Can acheive throughput of millions of messages/second and fault tolerance with replication.

**Logs compared to traditional messaging**

To achieve load balancing across consumers, the broker assigns entire partitions to nodes in the consumer group. Each client then consumes *all* the messages in the partitions it has been assigned, sequentially in a single thread. Downsides:

* The number of nodes sharing the work of consuming a topic is at most the number of log partitions in that topic, because messages within the same partition are delivered to the same node.
* If a single message is slow to process, it holds up the processing of subsequent messages.

If messages are expensive to process and you want to parallelize processing on a message-by-message basis, and ordering is not important, use JMS/AMQP message broker. In situations with high message throughput, where each message is fast to process and message ordering is important, the log-based approach works well.

**Consumer offsets**

Consuming a partition sequentially makes it easy to tell which messages have been processed: all messages with an offset less than a consumer's current offset have already been processed. No acks needed (higher throughput). If a consumer node fails, another node in the group is assigned to the failed consumer's partitions, and it starts consuming messages at the last recorded offset.

**Disk space usage**

The log is divided into segments and old segments are deleted or moved to archive storage. Logs implement a *circular buffer* or *ring buffer* that discards old messages when it gets full.

**When consumers cannot keep up with producers**

Log-based approach uses a form of buffering with a large, fixed-size buffer (limited by disk space). If a consumer falls too far behind, it may miss old messages that are not retained on disk by the broker.

One consumer falling behind does not affect another. You can also experimentally consume a production log for dev/testing without disrupting production services.

**Replaying old messages**

In a log-based system, reading messages does not have a deletion side effect like in AMQP/JMS-sytle brokers. Thus you can start reading at different points in time, with different processing code: which allows for more experimentation and easier recovery from bugs.

### Databases and Streams

An event could be a *write to a database*. Ex: a replication log is a stream of database event, produced by the leader that followers apply. *State machine replication* in total order broadcast is another case of event streams.

#### Keeping Systems in Sync

Most applications need to combine multiple different storage technologies to satisfy needs, each with its own copy of the data. This data needs to be kept in sync. Sometimes this is done by an ETL process or batch job. If this is too slow, an alternative is *dual writes*, where application code explicitly writes to multiple systems when data changes.

Dual writes have the problem of race conditions and fault tolerance if one write fails and one succeeds.

#### Change Data Capture

*Change data capture (CDC)* is the process of observing all data changes written to a database and extracting them in a form in which they can be replicated to other systems: changes can be made available as a stream immediately as they are written.

**Implementing change data capture**

CDC makes one database (the system of record) the leader and turns the others (derived data systems) into followers. We use a log-based message broker for transporting change events since it preserves the ordering of messages. CDC is usually asynchronous.

**Initial snapshot**

Often it makes sense to start with a snapshot of the db that corresponds to a known position/offset in the change log. This way you don't have to keep all changes forever which might require too much disk space.

**Log compaction**

*Log compaction* is an alternative to the snapshot process. The storage engine can periodically (in the background) look for log records with the same key, throw away duplicates, and keep only the most recent update for each key. A *tombstone* is used to indicate a key was deleted (and can be removed during compaction).

To rebuild a derived data system: can simply start at offset 0 in a compacted log and scan over all messages.

This is supported by Kafka and allows message broker to be used for durable storage.

**API support for change streams**

Some newer dbs support CDC as a first-class interface, e.g. RethinkDB, Firebase, and CouchDB.

#### Event Sourcing

*Event sourcing* is similar to CDC, but events are stored at a different layer of abstraction:

* Application logic is explicitly built on basis of immutable events that are written to an append-only event log. Events are designed to reflect things that happened at the application level, rather than low-level db state changes.

Event sourcing makes it easier to evolve the application over time, helps with debugging by making it easier to understand why something happened, and guards against application bugs.

Example: storing the event "student cancelled their course enrollment" instead of the side effects "one entry was deleted from the enrollments table and one cancellation reason was added to the student feedback table" (which embeds assumptions about the data).

**Deriving current state from the event log**

Applications that use event sourcing need to be able to deterministically transform the log of events (how the data was *written*) into the current state for showing to the user.

Log compaction is not possible as in CDC because later events do not necessarily override earlier events and events are modeleted at a higher level. Therefore applications that use event sourcing usually have a mechanism for storing snapshots of current state.

**Command and events**

When a request from a user first arrives, it is a *command*: it could still fail, e.g. if it violates a constraint (ex: registering a username, booking a seat on a plane). After the check has succeeded, the *event* is generated. It becomes a *fact*.

#### State, Streams, and Immutability

Databases store the current state of the application: this representation is optimized for reads. An append-only log of immutable changes, the *changelog*, represents the evolution of a mutable state over time.

**Advantages of immutable events**

Like an accountant keeping a *ledger*, an append-only log of immutable events allows auditability, which is beneficial for many systems. E.g. financial systems or making it easier to diagnose buggy code that writes bad data to the db.

Immutable events also capture more info than the current state. Example: a user adds an item to their cart then removes it. This is recorded in an event log, but lost in a database that deletes the item.

**Deriving several views from the same event log**

By separating mutable state from the immutable event log, you can derive several different read-oriented representations from the same log of events: like having mutliple consumers of a stream.

Having an explicit translation step from an event log to a database makes it easier to evolve your application over time: you can use the event log to build a separate read-optimized view and run it alongside the existing systems. Running new and old systems side by side is easier than performing a schema migration.

You gain flexibility by separating the form in which the data is written from the form it is read and by allowing several different read views. This is known as *command query responsibility segregation (CQRS)*.

**Concurrency control**

The biggest downside of event sourcing and CDC is that the event log consumers are usually asynchronous so you may not be able to "read your own writes".

One solution is to perform updates to the read view synchronously with appending to the event log. This requires a transaction, so you need to keep the event log and the read view in the same storage system or you need a distributed transaction across different systems.

On the other hand, event sourcing makes multi-object transactions less needed (the event can be atomic). If the log and application state are partitioned the same way, then a single-threaded log consumers needs no concurrency control for writes.

**Limitations of immutability**

If there is significant churn in the database, i.e. a high rate of updates and deletes on a relatively small database, the immutable history may grow prohibitively large.

There are also cases where you need data to be truly deleted (as if it was never written), for privacy regulation reasons, for example. This is known as *excision* or *shunning*.

### Processing Streams

Options for processing streams:

1. Take the event data and write it to a derived data sytem (e.g. db, cache, search index).
1. Push the events to users/humans, e.g. email, push notif, stream to dashboard.
1. Process one or more input streams and produce one or more output streams. Streams may go through a pipeline of several processing stages before going to an output.

A piece of code that processes a stream is known as an *operator* or *job*.

#### Uses of Stream Processing

Stream processing is classically used for monitoring & alerting if a certain thing happens, for example:

* Fraud detection, trading systems, manufactoring systems, military intelligence.

**Complex event processing**

*Complex event processing (CEP)* (from the 1990's) uses a high-level declarative query language like SQL to describe the patterns of events that should be detected. The queries are submitted to the processing engine that consumes the input streams. When a match is found, a *complex event* is emitted.

The relationship between queries and data is reversed compared to normal databases: queries are stored long-term and events from input streams continuously flow past. In a normal db a query is forgotten by the db after it has been processed.

**Stream analytics**

*Analytics* on streams is for computing aggregations and statistical metrics over a large number of events, for example:

* Measuring the rate of some event.
* Calculating the rolling average of a value over a time period.
* Comparing current statistics to previous time intervals (e.g. to detect trends or alert).

Such statistics are usually computed over some time interval, known as a *window*.

Stream analytics sometimes use probabilistic algorithms (e.g. Bloom filter, percentile estimation) as an optimization. But stream processing is not inherently approximate.

Frameworks: Apache Storm, Spark Streaming, Kafka Streams.

**Maintaining materialize views**

A stream of changes to a database can be used to keep derived data systems, such as caches, search indexes, and data warehouses up to date with a source database. In these cases, it is not sufficient to consider only events within a time window: building a *materialized view* potentially requires *all* events, apart from obsolete events that may be discarded by log compaction.

**Search on streams**

We sometimes need to search for individual events based on complex criteria, such as full-text search queries. Use case examples: 

* Media monitoring the news for stories mentioning a certain topic/company/product.
* Users of a real estate website being notified when a new property matching their criteria is posted.

This is done by formulating a search query in advance and then continually matching the stream against this query.

Conventional search engines first index the documents and then run queries over that index. Searching a stream turns the processing around: the queries are stored, and the documents run past the queries.

**Message passing and RPC**

Message-passing systems are an alternative to RPC, for example in the actor model, but they aren't the same as stream processors:

* Actor frameworks are primarily a mechanism for managing concurrency vs. stream processing which is for data management.
* Communication between actors is one-to-one and ephemeral vs. durable and multi-subscriber.
* Actors can communicate in arbitrary ways (including cyclic) vs acyclic pipelines for stream processing.

But there is some crossover, e.g. Apache Storm *distributed RPC*.

#### Reasoning about Time

Stream processors often deal with time, especially for computing analytics over a window. Many stream processing frameworks use the local system clock on the processing machine (the *processing time*) which can cause problems if there is processing lag.

**Event time versus processing time**

There are many reasons why processing may be delayed: queueing, network faults, performance issues on the broker or processor, a consumer restart, or reprocessing old events.

Message delays can lead to unpredictable ordering of messages: for example two web requests from the same user that are processed on different servers - a stream processor could see them as out of order.

Confusing event time could lead to bad data, e.g. what *looks* like a spike in request rate/QPS.

**Knowing when you're ready**

A tricky problem when defining windows in terms of event time is that you can never be sure when you have received all of the events for a particular window or whether there are *straggler* events still to come. Two options:

1. Ignore straggler events, as they are probably a small percentage of events. You can track and alert on the number of dropped events itself as a metric.
1. Publish a *correction*: an updated value for the window with stragglers included.

**Whose clock are you using, anyway?**

Events can be buffered at several points in time, e.g. a mobile app reporting usage metric events to a server. The app could be used offline and buffer the events locally before coming back online and sending them to a server. These could appear as extremely delayed stragglers, so we should really use the device's local clock. But we can't trust the clock on a user-controlled device. To adjust, log three timstamps:

1. The time at which the event occured, according to the device clock.
1. The time at which the event was sent to the server, according to the device clock.
1. The time at which the event was received by the server, according to the server clock.

By subtracting the 2nd from the 3rd, we can estimate the offset between the device clock and the server clock.

**Types of windows**

* *Tumbling window*: has a fixed length and every event belongs to exactly one window, e.g. a 1-min window and all events with timestamps between 10:03:00 and 10:03:59 are grouped into one window, and events between 10:04:00 and 10:04:59 another.
* *Hopping window*: has a fixed length, but allows windows to overlap to provide smoothing. E.g. a 5-min window with a hop size of 1-min contains events between 10:03:00 and 10:07:59 and the next window covers 10:04:00 and 10:08:59.
* *Sliding window*: contains all the events that occur within some interval of each other, e.g. 5 minutes.
* *Session window*: has no fixed duration, but ends after some period of inactivity. E.g. all events for the same user that occur closely together in time until the user has been inactive for 30 min.

#### Stream Joins

There are three types of joins in stream processing: *stream-stream joins*, *stream-table joins*, and *table-table joins*.

**Stream-stream join (window join)**

Example: calculating search result click-through rate by recording search query events and click events.

To implement this, a stream processor must maintain *state*, e.g. all events that occurred in the last hour, indexed by session ID. When a search or click event occurs, it is added to the appropriate index, and the stream processor also checks the other index to see if another event for the same session ID already arrived. If there is a matching event, the processor emits an event saying which search result was clicked. If the search event expires without seeing a matching click, emit an event saying which search results were not clicked.

**Stream-table join (stream enrichment)**

Example: user activity events need to be *enriched* with profile information from the user db.

To do this, the stream process needs to look at one activity event at a time, look up the event's user ID in the db. But we shouldn't query a remote db. Instead, we should load a copy of the db into the stream processor so it can be queried locally.

The stream processor needs to keep this db up-to-date, though. This can be done with CDC.

**Table-table join (materlized view maintence)**

Example: keeping a per-user cache of their Twitter timeline.

To implement, need streams of tweets (send and delete) and follows/unfollows. The stream process needs to maintain a db containing the set of followers for each user so it knows which timelines need to be updated.

i.e. the stream process maintains a materialized view for a query that joins two tables (tweets and follows).

**Time-dependence of joins**

What if there is a time dependence between a stream and the data it needs to be joined with? Example: calculating tax rate for invoices when the tax rate occassionally changes.

This issue is known as *slowly changing dimension (SCD)* and it is solved by using a unique identifier for a particular version of the joined record. This makes the join deterministic but makes log compaction not possible.

#### Fault Tolerance

In batch processing, even if a task fails, it can be transparently retried, so even if a record was processed multiple times, the visible effect in the output is the same as if it was processed once. This is known as *exactly-once semantics* or, better, *effectively-once*.

**Microbatching and checkpointing**

One solution for fault tolerance in streaming is to break the stream into small blocks and treat each block like a mini batch process: *microbatching*. 

You can also generate rolling checkpoints of state and write them to durable storage. A crashed stream operator can restart from the most recent checkpoint.

**Atomic commit revisited**

To provide exactly-once processing in the presence of faults, all outputs and side effects of processing an event take effect *if and only if* the processing is successful. E.g. message sent downstream, db writes, external messages (email or push), acks of incoming message.

These should all happen atomically or none at all.

**Idempotence**

An *idempotent* operation is one you can perform multiple times and has the same effect as if you performed it only once.

You can make a non-idempotent operation idempotent by including extra metadata. e.g. in Kafka, every message has a monotonically increasing offset. Including that offset in a write to an external database can tell you whether an update has already been applied.

## Chapter 12: The Future of Data Systems

### Data Integration

There is unlikely to be one piece of software that is suitable for *all* the different read patterns for your data.

#### Combining Specialized Tools by Deriving Data

For example, it is common to need to integrate an OLTP db with a search index, cache, analytics systems, etc.

**Reasoning about dataflows**

If you can, it's preferrable to write all data first to a system of record, capturing changes, and applying them in the same order (via CDC) to derived data systems. This is better than concurrently writing directly to multiple datastores at the same time, which can cause conflicting writes.

**Derived data versus distributed transactions**

The classic approach for keeping different data systems in sync involves distributed transactions (atomic commit and 2PC). CDC & event sourcing use a log for ordering vs locks for mutual exclusion in distributed transactions. Distributed transactions use atomic commit vs deterministic retry & idempotence in log-based systems to make sure events take place exactly once.

Transaction systems usually provide linearizability (read your own writes) whereas derived data systems are often updated asynchronously.

**The limits of total ordering**

Limitations of a single-leader replication system (that de facto has a totally ordered log): 

* Cannot partition if needed because of need for a *single leader node*.
* Cannot handle *geographically distributed* datacenters (which each usually have their own leader).
* *Microservices* usually each have their own storage, with no durable state shared.
* Clients with client-side state or offline operation.

**Ordering events to capture causality**

Sometimes, two events have a *causal dependency*, e.g. unfriend and then post on your facebook wall. Notifications for this are effectively a join between posts and friends. There is no easy solution. Some ways to partially solve:

* Logical timestamps when total order broadcast is not feasible.
* Log an event to record the state of the system the user saw before making a decision and give that event a unique ID that later events can reference.
* Conflict resolution algorithms for maintaining state, but not if the actions have external side effects.

#### Batch and Stream Processing

The goal of data integration is to make sure that data ends up in the right form in all the right places. Batch and stream processors are starting to become less distinct over time.

**Maintaining derived state**

Batch processing is functional: it encourages deterministic, pure functions, whose output depends only on the input and has no side effects and immutable inputs.

Unlike synchronously-updated systems like relational dbs with secondary indexes, asynchrony makes systems based on logs robust: faults in one part of the system are contained locally.

**Reprocessing data for application evolution**

Batch and stream processing are useful for deriving and maintaining new views on an existing dataset. Reprocessing existing data is a good mechanism for evolving to support new features and changing requirements.

Derived views allow *gradual* evolution: instead of performing a sudden schema change, you can maintain the old and new schema side by side and shift users over time.

**The lambda architecture**

The *lambda architecture* uses a combination of batch and stream processing: the stream processor consumes the events and quickly produces an approximate update to the view; the batch processor later consumes the *same* events and produces a corrected version of the derived view.

**Unifying batch and stream processing**

Unifying batch and stream processing without the downsides of lambda architecutre requires:

* The ability to replay historical events through the same processing engine that handles the recent event stream.
* Exactly-once semantics for stream processors (e.g. ensuring the output is the same as if no faults had occurred).
* Tools for windowing by event time, not processing time.

### Unbundling Databases
