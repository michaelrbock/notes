# Databases

[Bradfield](https://bradfieldcs.com/courses/databases/)

-

### Notes on [Architecture of a Database System](http://db.cs.berkeley.edu/papers/fntdb07-architecture.pdf) paper

RDBMS (Relational Database Management System) has 5 main parts (in order they would handle a query):

* Client Communications Manager
  * Sets up connection with the calling client.
* Process Manager
  * Assigns a *thread of computation* to a given commad. Also handles *admissions control*: whether to begin processig query now or wait for more resources to become available.
* Relational Query Processor
  * Executes query. Compiles SQL into *query plan*.
  * The compiled query plan is handled by thte *plan executor* which consists of many *operators* (algorithm implementations, e.g. for joins, selection, projection, aggregation, sorting).
* Transactional Storage Manager
  * Handles calls from the operator which requests data from the DB.
  * Manages reads and manipulation (create, update, delete) calls.
  * Includes algorithms and data structures for organizing and accessing data on disk (*access methods*) like tables and indexes.
  * Includes a buffer management module that decides when to transfer data from memory to disk.
  * Before accessing data, locks are acquired from the lock manager.
  * Must ensure ACID properties of transactions via transaction management code.

After the Transactional Storage Manager acts, the system "unwids the stack": access methods return control to executor operators which compute results which are placed in a buffer for the Client Communications Manager which ships them back to the caller. Then each system cleans up.

* Shared Components and Utilities
  * Memory Manager: for dynamic memory allocation.
  * Catalog Manager: used by query processor for authentication, parsing, query optimization.

## CS 186 - Databases

[Berkeley](link)

### Class 1 - Intro

Today: we deal with distributed data that's spread across lots of computers. Patterns (also works if data is to big to fit in memory):

* Streaming
* Divide-and-Conquer

Simplifying assumption for big data processing: only dealing with *unordered* collections. Allows:

* Order things to our liking (e.g. cache locality)
* Work in batches (e.g. based on data affinity, OK to postpone data)
* Can tiolerate non-deterministic orders (e.g. for parallelism)

#### Streaming

Example: read from 100GB file, compute f(x) for each record, and write out results (can't read whole file intp RAM and want to minimize read/write calls):

1. Read chunk from input into an *input buffer*.
1. Write f(x) for each record into an *output buffer*.
1. When *input buffer* is consumed, read another chunk.
1. When *output buffer* is full, write it out.

-> Reads and writes are *not* coordinated (can have 10x reads to writes for example).

Easy to Parallelize. Same thing UNIX Pipes do.

#### Time-Space Rendezvous

Need multiple input chunks in memory at the same time. Use Divide and Conquer (*out of core* algorithms):

* Allocate `B` chunks of RAM.
* Use `1` chunk for reads.
* Use `1` chunk for writes.
* Have `B-2` chunks as space for rendezvous.
* Phase 1: "streamwise" divide data into `N/(B-2)` megachunks (process)
  * *conquer* each and write to disk
* Phase 2: streaming algorithm over *conquered megachunks*
  * the streaming must ensure rendezvous

Parallelize?

* Phase 1+: partition data for rendezvous in space.

### Class 2 - Sorting and Hashing

Why Sort?

* Rendezvous (get items in memory at the same time)
  * Eliminate duplicates
  * Summarize groups of items
* Ordering
* Fundamental to *sort-merge-join* algo and building *tree indexes*

Problem: sort 100GB of data with 1GB of RAM.

Storage hierarchy (from big & slow to small & fast): Disk < SSD < RAM < On-Board Cache < On-Chip Cache < Registers.

Disk:

* Seek time (~2-4ms) and rotational delay (~2-4ms).
* Key to lower I/O cost: reduce seek/rotation delays.
* There's a concept of 'next' (same track->same cylinder->adjacent cylinder), so arrange files sequentially on disk.
* Can also readahead/pre-fetch.

Flash (SSD):

* ~100x faster than disk for random reads.
* Random reads are just as fast as sequential reads.
* Random writes slightly slower than sequential writes.
* Write endurance issues.
* Improve performance over Disk and also improves *performance variance*.

Storage trends:

* Data size isn't always actually that big, e.g. 2009 English Wikipedia ~= 14GB.
* But data sizes grows faster than Moore's Law.

Double Buffering (performance improvement over out-of-core streaming):

* Main thhread runs *f(x)* on one pair of I/O buffers.
* 2nd "I/O thread" fills/drains unused I/O buffers.
* Main thread ready for a new buffer? Swap!

Sorting & Hashing:

* Given:
  * A file `F`:
    * containing a multiset (could have duplicates) of records, `R`
    * consuming `N` blocks of storage
  * Two "scratch" disks
    * Each with >> `N` blocks of free storage
  * A fixed amount of space in RAM
    * memory capacity equivalent to `B` blocks of disk
* Sorting:
  * Produce an output file, `Fs`
    * with contents `R` *sorted in order by a give criteria*
* Hashing
  * Produce an output file, `Fh`
    * with contents R, *arranged on disk so that no 2 records that are equal are seperated by anoter record*
    * i.e. matching records are always "sorted consecutively"

Sorting - 2-Way (naive):

* Two-Way External Merge Sort
* Pass 0 (conquer):
  * read a page, sort, write
* Pass 1,2,3... (merge)
  * Streaming merge pairs of runs into runs twice as long
  * Keep doubling size of runs
* `N` pages in file, `log2(N) + 1` passes of the entire data set, so `2N * log2(N) + 1` measured in disk transfered

General External Merge Sort:

* To sort a file with `N` page using `B` buffers:
  * Pass 0: use `B` buffers, produce `N / B` sorted runs of K pages
  * Pass 1,2,...: merge `B-1` runs
* Number of passes: `1 + logB-1(N/B)`
* Cost: `2N * (# passes)`

How big of a table can we sort in two passes?

* Each "sorted run" after Phase 0 is size `B`
* Can merge up to `B-1` sorted runs in Phase 1
* Answer: `B(B-1)`
  * Sort `N` pages of data in about `√(N)` space

Internal Sort (heapsort):

* Keep two heaps in memory, `H1` and `H2`

* ```
read B-2 pages of records, inserting into H1
while (records left):
  min = H1.removemin()  # write min to output buffer
  if (H1 not empty):
    read in a new record r
    if (r < m): H2.insert(r)
    else:       H1.insert(r)
  else:
    H1 = H2
    H2.reset()
    start new output run
```

* If data is already sorted: only takes 1 Pass
* If data is in reverse order: keep getting runs of `B-2`
* Avg length of run: `2(B-2)`
* Quicksort is usually faster, but longer runs often means fewer passes

Alternative: Hashing

* Many times we don't require order
  * e.g. removing duplicates
  * e.g. forming groups
* Often just need to *rendezvous* matches
* Hashing does this and is cheaper than sorting!

Divide:

* Streaming Partition (divide): use a hash fxn `hp` to stream records to disk partitions
  * All matches rendezvous to same partition
  * Streaming alg to create partions on disk: spill partitions to disk via output buffers

& Conquer:

* ReHash (conquer): read partitions into RAM hash table one at a time using hash fxn `hr`
* Then go thru each bucket
