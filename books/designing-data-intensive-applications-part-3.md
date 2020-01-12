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

If the inputs to a map-side join are partitioned in the same way, then the hash join approach can be applied to each partition independently.
