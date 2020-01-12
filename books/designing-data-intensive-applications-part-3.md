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
