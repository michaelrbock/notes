# Bigtable: A Distributed Storage System

[Paper](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)

Features/properties:

* Does not support full relational model.
* Clients have control over data layout and format.
* Data is indexed by row and column names which are arbitrary strings.
* Data is uninterpreted strings.
* Clients control data locality through schema and whether to serve data from memory or disk.


### Data Model

A Bigtable is a sparse, distributed, persistent multi- dimensional sorted map. The map is indexed by a row key, column key, and a timestamp; each value in the map is an uninterpreted array of bytes:

`(row:string, column:string, time:int64) → string`

#### Rows

* Reads/writes to a single row key are atomic (regardless of number of columns involved).
* Data is maintained in lexicographical order by row key.
* Row ranges are dynamically partitioned into *tablets*.
    * Reads of short row ranges are efficient.

#### Column Families

* Column keys are grouped into *column families*.
* A column key is named using the following syntax: *family:qualifier*.
    * E.g. *language:ID* or *anchor:link*

#### Timestamps

* Each cell in a Bigtable can contain multiple versions, indexed by 64 bit timestamp.
    * Can be automatically set as real time in microseconds or set explicityly by client.
* Stored in decreasing timestamp order, most recent read first.
    * Can automatically garbage collect versions older than *X days* or keep only last *n* versions.

#### Building Blocks

* *SSTable*: persistent, ordered immutable map of keys to values.
    * Each SSTable contains a sequence of blocks.
    * An in-memory block index is used to locate blocks. Simply binary search this to find where on disk the block is (or store SSTable in memory).
* *Chubby*: highly-available distributed lock service.
    * Has 5 replicas, one of which is the master, and uses Paxos algorithm to keep replicas consistent.

## Implementation

Consists of: a client library, a master server, several tablet servers (dynamically added/removed).

* The master server handles: assigning tablets to tablet servers, detecting new servers, balancing tablet-server load, scehma changes.
* Each tablet server managers a set of (~10-1k) tablets. Handles read/writes and splits tablets when too large.
* Clients communicate directly with tablet servers for reads/writes. Clients do not usually communicat with master because they do not rely on it for tablet location information.

### Tablet Location

* Three-level hierarchy in a B+-tree.
* Chubby file -> *root tablet* (never split, contains *METADATA* tablet) -> *METADATA tablets* (stores location as tablet's table and end row) -> user tables
* Client library caches location of tablets (and does some pre-fetching).

### Tablet Assignment

* Each tablet is assigned to one tablet server at a time, handled by master server, which knows this information.
* When a tablet server starts up it acquires an exclusive lock on a file in a Chubby *servers directory* which the master server monitors.
    * A tablet server stops serving tablets if it loses its lock (e.g. from network failure).
    * A tablet server will attempt to re-acquire the lock, but will kill itself if can't.
    * When tablet server is taken down, it releases its lock quickly.
* The master server is responsible for detecting tablet servers are serving tablets.
    * It periodically asks each tablet server for the status of its lock.
    * If the tablet server reports it has lost its lock or can't be reached, the master attempts to acquire its lock and deletes its file (tablet server is dead or can't reach Chubby).
        * Then, the master can move tablets to unassigned.
    * Master server kills itself if it can't reach Chubby.
* On startup:
    1. Master server acquires unique *master* lock.
    1. Scans server directory in Chubby to find live servers.
    1. Communicates with every live tablet server to discover tablet assignments.
    1. Scans METADATA table to learn of unassigned tablets.

### Tablet Serving

* Updates are written to a commit log, newer ones in a *memtable*, older in SSTables.
    * To recover a tablet, a server can find the SSTables from the METADATA table and the set of redo points to reconstruct the memtable and apply updates.
* Writes, if well-formed and authorized, are written to the commit log (possibly grouped), once committed, inserted into memtable.
* Reads, if well-formed and authorized, look through a view of merged memtable and SSTables (both sorted, so efficient).
* Read/writes can happen during tablet split/merge.

### Compactions

* *Minor compaction*: memtable is frozen and written to a new SSTable.
* *Merging compaction*: reads the contents of a few SSTables and the memtable, and writes out a new SSTable.
* *Major compaction*: A merging compaction that rewrites all SSTables into exactly one SSTable.
    * Temporary SSTables may contain deleted tombstones, but major compaction SSTables don't have deleted data.

### Refinements

#### Locality Groups

* Clients can group multiple column families together into a locality group.
    * A separate SSTable is generated for each locality group in each tablet.
    * This enables more efficient reads.
* Locality groups can be specified to be stored in-memory (used for METADATA table).

#### Compression

* Each SSTable block is compressed separately.
    * Lose some space by compressing each block separately, benefit that small portions of an SSTable can be read without decompressing the entire file.

#### Caching for read performance

* Two levels of caching:
    * Scan cache: for key-value pairs from SSTable to tablet server code.
        * Good for reading the same data repeatedly.
    * Block cache: lower-level cache that caches SSTables blocks that were read from file system.
        * Good for applications doing reads of data that is close to the data they recently read.

#### Bloom filters

* Optionally used to avoid looking back over many SSTables.

#### Commit-log implementation

* Commit logs are not stored by tablet (would be too many disk seeks for file writes), instead append mutations per tablet server, co-mingling mutations for different tablets into one log file.
    * Performance benefits during normal operation, but complicates recovery.
        * Commit log entries are sorted by `table, row name, log sequence number`.

#### Speeding up tablet recovery

* When the master moves a tablet from one server to another, the original server performs two minor compactions.

#### Exploiting immutability

* SSTables are immutable, so only memtable needs to be locked for concurrency control.
* The master removes obsolute SSTables via mark and sweep garbage collection.
