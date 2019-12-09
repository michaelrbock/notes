# Designing Data-Intensive Applications - Part I

[Goodreads](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications)

-

## Chapter 1 - Reliable, Scalable, and Maitainable Data Systems

#### *Reliability*: The system should work correctly, even in the case of hardware/software/human error.

* Includes correctness, edge case handling, speed, security.
* Types of *faults* (one component breaking, that could lead to *failures*, where the whole system is down):
  * Hardware (usually unrelated, but common).
  * Software (often cascading, can lie dormant until some unusual circumstance); Mitigations:
	  * Careful thinking,
	  * Testing,
	  * Process isolation,
	  * Measuring, monitoring, analyzing production.
  * Human; Mitigations:
	  * Well-abstracted APIs,
	  * Sandboxes,
	  * Testing (unit, itegration, manual),
	  * Allow fast/easy rollbacks,
	  * Telemetry (aka monitoring),
	  * Good management and training (good luck).

#### *Scalability*: As the system grows, there should be reasonable ways to deal with that growth.

* Don't say: "X is scalable", "Y doesn't scale".
* Yes: "If the system grows in a particular way, what are our options for coping?".
* *Load parameters*: e.g. requests/second, cache hit rate, read/write ratio, average/peak.

**Example: Twitter Timeline, two approaches**:

1. On timeline load, get tweets from all users the current user follows, merge them and sort by time.
1. Maintain a cache for each users' timeline. When someone tweets, insert that into each of their followers' timeline caches.

* Two questions to assess performance:
  1. When you increase a load parameter and keep system resources the same, how does that affect performance?
  1. How much would you need to increase resources to keep performance the same if you increased a load parameter?
* Describing performance: e.g. throughput for a batch system or response times (use percenntiles, e.g. *p50, p95, p99*). The *tail latencies* are often from the most valuable users. SLO/SLAs often defined by percentiles.
* *Head of line bocking*: slow requests block the server from responding to subsequent requests.
* *Tail latency amplification*: the slowest backend call (even when made in parallel) will determine end-user response time.
* Types of scaling:
  * *Vertical*
  * *Horizontal*
  * *Shared-nothing*: distributing load across multiple machines
  * *Elastic*: automatically increases resources under load

#### *Maintainability*: People in the future should be able to work on the system *productively*.

* *Operability*: allow operations to keep system running smootly.
  * Monitoring, security, configuration changes, maintence, documentation/playbooks, good defaults, self-healing, predictability.
* *Simplicity*: easy for new engineers to understand the system.
  * Remove *accidental* (not inherint in the problem the software solves, but only from the implementation) complexity by using good *abstraction*.
* *Evolvability*/*extensibility*: easy to make changes to system as requirements change. 

## Chapter 2 - Data Models and Query Languages

#### Relational Models vs. Document Model

Reasons for NoSQL adoption:

* Need for greater scalability (very large datasets or high write throughput).
* F/OSS > commercial.
* Specialized query operations.
* More dynamic/expressive data model.

*Impedence mismatch*: disconnect between relational data model and OO application code.

Example: Modeling a LinkedIn Resume:

* `user_id`, `first_name`, `last_name` as columns in `users` table.
* positions, education, contact info in separate tables with foreign keys.
    * or: put mutli-value data in single row with structured datatypes in SQL.
    * or: encode as JSON/XML document in text column.
* or: use an entirely self-contained JSON document in a document-oriented database.
    * Pros: reduces impedence mismatch, scehma flexibility, better *locality* (no joins or muiltiple queries), easily represents tree structure of data.

#### Many-to-One and Many-to-Many Relationships

Choice between storing IDs (e.g. `region_id` or `industry_id` and plain-text strings, e.g. "Greater Seattle Area" or "Philanthropy"). Pros of using IDs (and a drop-down list/autocompleter UI):

* Consistent style/spelling.
* Avoiding ambiguity across several cities with same name.
* Ease of updating - only need to change in one place.
* Localization support.
* Better search.
* Don't duplicate human-readable data (just the IDs).
* IDs can be immutable even if info it identifies changes.

This is called data *normalization* and relies on *many-to-one* relationships vs. *duplicating* data.

Cons:

* If the document DB itself does not support joins, must emulate in application code by making multiple queries.
* Data tends to become more interconnected as features are added, which often have *many-to-many* relationships.

#### Relational vs. Document Databases Tradeoffs

* Use a document model if your application data has a document-like strucutre (i.e. a tree of one-to-many relationships, where typically the entire tree is loaded at once).
    * This avoids *shredding* in relational model which splits a document into multiple tables.
    * Avoid *too*-deeply nesting data.
    * Be careful of poor support for joins. 
        * If your application has many-to-many relationships, the document model is less appealing.
        * De-normalizing helps, but application needs to keep denormalized data consistent.
        * Joins can be emuluated in application code, but is usually slower than DB joins.

* Document databases are usually *schema-on-read* (aka *schemaless*, implicit schema, interpreted when read), vs. *schema-on-write* in relational DBs.
	* Analogy to dynamic/runtime vs. compiled  type checking in programming languages.
* Schema changes in dociument model usually involve `if/else` statements in code vs. `ALTER TABLE`/`UPDATE` migrations in relational model.

**Data Locality**

* Documents are stored continuously and have performance advantage of *storage locality*.
* Only applies if you need much of the document at the same time: wasteful if you only need a part of it at a time or need to do writes (the whole document needs to be re-written).

**Convergence of Document & Relational Models**

* Models are getting more similar over time: e.g. support for JSON in SQL DBs and joins in some document DBs.

### Data Query Languages

#### Declarative Languages

* SQL is a *declarative* language which specifies thhe pattern of data you want (as opposed to a *imperative* programming language where you specify *how* to achieve goal). 
* Declarative languages hides implementation details which allows performance improvements in database engine.
* Declarative languages lead to parallel execution.
* CSS is similarly declarative.

#### MapReduce Querying

* MonogoDB implements a MapReduce pattern which mixes imperative (*pure* `map` and `reduce` functions) with a declarative query language.
* Distributed SQL queries *can* be implemented as a MapReduce pipeline.

### Graph-Like Data Models

* Useful for data with common many-to-many relationships, e.g.
    * Social graphs, Web pages with links, Road or rail networks.
* Verticies do *not* have to be *homogeneous*: you can represent many different types of objects in a single graph.

#### Property Graphs

Each vertex has:

* A unique ID
* A set of outgoing edges
* A set of incoming edges
* A collection of (key-value) properties

Each edge has:

* A unqiue ID
* The *tail vertex* (at which it starts)
* The *head vertex* (at which it ends)
* A label to describe thhe relationship between the two verticies
* A collection of (key-value) properties

You could store this in two relational tables: verticies and edges. 

* This is much more flexible than a normal relational DB for data modeling.
* Can traverse the graph with `INDEX`es for tail and head verticies.
* Can store several different kinds of information in one graph by having labels for different types of relationships.

#### Cypher Query Language

For Neo4j: a *declarative* language for modeling and querying graph datastore.

#### Graph Queries in SQL

Can use `WITH RECURSIVE` SQL syntax, but it's much more complicated than Cypher's declarative statement.

#### Triple Stores and SPARQL

All information is stored in three-part statements: (*subject*, *predicate*, *object*), e.g. `(Jim, likes, bananas)`.

Can store properties, e.g. `(lucy, age, 33)` or relationships, e.g. `(lucy, marriedTo, bob)`.

**Semantic Web**

A project to represent web sites in a machine-readable format in addition to human-readable. Uses RDF: Resource Description Framework.

## Chapter 3 - Storage and Retrieval

### Data Structures

* *log* = append-only sequence of records (very fast writes, `O(n)` retrievals).

* *index* = an *additional* structure that stores metadata on the side helping to locate data.
    * Can add and remove indexes without affecting contents of primary data, only affects query performance.
    * Incurs write overhead.
    * Chosen by application developer.

#### Hash Indexes

* Simply store a hash map in memory of `{key : byte offset in data file}`, while appending to data file for each write/update.
* Well-suited for workloads with not too many keys (can keep in memory), but where each key's value is updated frequently (ex: updating number of times a video has been played).
* Can break log into segment files of a certain size and perform *compaction* on closed segments. Compaction means throwing away duplicate keys in log and keeping only the most recent update for each key.
    * Can also merge across segments (since segments are never modified). Merging and compaction can happen in background threads while serving reads and writes as normal until finished.
    * Each segment has it's own hash map with file offsets.
* Additional details:
    * Binary format for segment files, not CSV.
    * Deleting records requires a special deletion record (a *tombstone*): can discard previous values when merging.
    * Crash recovery requires re-creating in-memory hash map (can snapshot them to disk).
    * Partially-written records (mitigate with checksums in case of crash mid-way thru write).
    * Only one thread for writes, multiple for reads.

* Limitations:
    * Hash table must fit in memory.
    * Range queries are not efficient.

#### SSTables and LSM Trees

* *SortedStringTable* = segement files of key-value pairs *sorted* by key.
    * Merging segments is efficient, similar to merge sort.
    * Only need to keep a (sparse) index to some keys to find any key in file, can scan between offsets for the rest (similar to binary search).
    * Can compress blocks before writing to disk and point index to start of block.
* Constructing SSTables:
    * Writes go to an in-memory balanced tree (red-black or AVL), called a *memtable*.
    * When the memtable gets bigger than some threshold, write it to disk as an SSTable file (easy because the tree is sorted by key). While this is writing to disk, writes go to a new memtable.
    * To serve a read request: first try to find in memtable, then in most-to-least recent segments.
    * Run merging/compaction periodically in background.
    * Also write to a log file in case of crash to restore memtable.

* [LevelDB](https://github.com/google/leveldb) (Google) and [RocksDB](https://rocksdb.org/) (Facebook) are key-value stores that use the algorithm above.
* Cassandra and HBase are inspired by Google's BigTable, where *SSTable* and *memtable* were introduced.
* Full text search is acheived by having a mapping of keys (search *terms*) to values (list of IDs of all documents containing that key/term: the *postings list*).
	* Lucene, an indexing engine for full-text search used by Elasticsearch and Solr uses this method for storing its *term dictionary*.

* *Bloom filters* are used to optimize reads where the key does not exist (to avoid searching back through all previous segments via disk reads).
* Compaction/merging strategies:
    * *Size-tiered*: newer and smaller SSTables are merged into older and larger ones.
    * *Leveled*: key range is split up into smaller SSTables and older data is moved into separate "levels", which allows compaction to proceed more incrementally and use less disk space.

* LSM-trees allow for fast range queries *and* high write throughput.

#### B-Trees

* Most common/standard indexing strategy (in relational DBs and others).
    * Break down database into fixed-size *blocks* or *pages* (traditionally 4KB in size). Keeps key-value pairs sorted by key.
    * Each page can can indentified using an address and one page can refer to another (like a pointer, but on disk).
        * Can use these page references to construct a tree of pages.
    * One page is the root. The page contains several keys and references to child pages. Each child is responsible for a contiguous range of keys (and keys surrounding the references indicate where the boundaries for those ranges lie).
    * See diagrams on pg. 80-81: Figures 3-6 and 3-7.
    * When looking up a key, follow references until you get to a *leaf page* (containing only keys inline or refernces to values for keys).
    * The *branching factor* is the number of references to child pages per page (typically several hundred).
    * To update a key: search for the leaf page containing that key and re-write the page.
    * To add a key: search for page whose range encompasses that key and add it. If there's not enough free space, split it into two half-full pages and the parent is updated.
    * A *balanced* B-Tree with `n` keys always has a depth of `O(log n)`. A four-level tree of 4 KB pages with a branching factor of 500 can store up to 256 TB.

**Making B-Trees Reliable**

* Writes happen via *overwritting*/*modifying* a page (not just appending like in LSM-trees).
* Some operations require overwrites to multiple pages.
    * This is dangerous because what if crash happens after only some pages have been written?
        * In order to be resilient to crashes, include a *write-ahead-log* (WAL or *redo log*). An append-only file where every modification must be written before it can be applied to the pages of the tree itself.
        * If coming back from a crash, can restore B-Tree to consistent state.
* *Latches* (lightweight locks) protect B-tree from concurrency 

**B-Tree Optimizations**

* Instead of WAL, use a copy-on-write scheme: modified page is written to a new location and parent reference is updated to point to the new page.
* Save space by storing abbreviated key.
* Try to keep leaf pages in sequential order on disk.
* Add additional pointers to tree (e.g. left/right pointers from leaf pages to siblings).
* *Fractal trees* borrow log-structured ideas to reduce disk seeks.

### Comparing B-Trees and LSM-Trees

* LSM-trees are typically faster for writes, whereas B-trees are faster for reads. Reads are slower on LSM-trees because they have to check several different data structs and SSTables at different stages of compaction.
* Performance needs to be tested on specific workload.

#### Advantages of LSM-Trees

* Lower *write amplification* (how many times data needs to be written to disk per write to the database over the database's lifetime).
    * Matters for performance in write-heavy applications.
    * Also important because sequential writes of an LSM-tree are much faster on magnetic disk than random writes.
* Compress better and produce smaller files on disk than B-trees (which have fragmentation).

#### Downsides of LSM-Trees

* Sometimes the compaction process will get in the way of concurrent access. A request may need to wait while the disk finishes an expensive compaction (affects higher percentiles).
    * B-trees can be more predictable.
* Disk's finite write bandwidth must be shared between initial write (memtable to disk) and background compaction operations.	
* Must watch out for high write throughput and misconfigured compaction where disk keeps growing and slows down reads and/or runs out of disk.

### Secondary Indexes

* Made by `CREATE INDEX` and crucial for performing joins efficiently.
* There may be many rows under the same index entry.
    * Can be solved by making each value in the index a list of matching row IDs or by making each entry unique by appending a row ID to it.

**Storing values within the index**

* Some indexes have values that include the row data itself while some store just a reference to the row elsewhere.
    * When storing references, the row itself is stored in a *heap file*.
* Storing the data directly in the index is known as a *clustered index*.
    * A *covering index* or *index with included columns* stores just *some* of the table's columns within the index.
    * Duplicating data requires additional storage and write overhead.

#### Multi-column indexes

* *Concatenated index* combines several fields into one key by just appending the columns together.
* *Multi-dimensional indexes* allow you to make range queries across multiple columns at once. E.g. *R-trees*.
    * Example: query for restaurants within rectangular area, i.e. 2 lattitudes and 2 longitudes.
    * Example: query for all dates with tempratures between 25 and 30 degrees in 2013 with 2D index on *(date, temprature)*.

**Full Text Search and Fuzzy Indexes**

* Lucerne can have a LSM-tree with in-memory finite state automaton, like a trie, to do fuzzy full text search.

#### Keeping everything in memory (*in-memory databases*)

* Stores everything in RAM for data that can fit.
    * Can also have a log of changes to disk for durability.
    * Performance advantage of avoiding encoding data structures to write to disk.
    * Allows implementations of interesting data structures like priority queues and sets (e.g. in Redis).

### Transaction Processing or Analytics?

* OLTP = *online transaction processing*
    * End-user initiated.
    * Reading a small number of records per query, fetching by key.
    * Random-access, low-latency writes.
    * Acessing latest state of data.
* OLAP = *online analytic processing*
    * Business analytics for business intelligence.
    * Aggregate over large number of records.
    * Write via bulk import (ETL) or event stream. Looking at history of data.
    * Can use same database or a separate *data warehouse*.

* Data is *Extract-Transform-Load (ETL)* from OLTP systems in data warehouse (read-only copy).
    * Indexes in OLTP will usually not be helpful for analytic queries.

#### Data Warehouse

* OLTP systems need to be highly-available and therefore want to avoid running ad hoc analytic queries on an OLTP database that may harm performance.
* Instead, *ETL* data from OLTP databases (either via periodic dump or continuous stream of updates) into *data warehouse*.
* Data warehouse can be optimized for analytic access patterns.

**Divergence between OLTP databases and data warehouses**

* Although both are accessed through a common SQL query interface, the internals can look quite different because they are optimized for different query patterns.

#### Schemas for Analytics

* Many data warehouses use a *star schema* (or *dimensional modeling*).
    * At the center of the schema is the *fact table*, usually as individual events.
    * Columns in fact table include attributes of event and foreign key refernces to other table, called *dimensional tables*.
    * Example: purchases in fact table; dates, product info, customer info in dimensional tables.
* *Snowflake schema*: dimensions are further broken down into subdimensions.
    * Example: Prodcut dimensional table has foreign keys to brand and category tables.
    * More normalized but more complicated to work with.
* Data warehouse tables are usually very wide.

### Column-Oriented Storage

* With trillions of rows and PBs of data in fact table, gets challenging to query efficiently.
    * For analytics, only query a subset of columns even if you have 100+ column-wide tables.
* *Column-oriented storage* stores all values from each *column* together (as opposed to *row-oriented* as in most OLTP databases).
    * Allows us to only load data from columns we are interested in.
    * Relies on each column file containing the row's data in the same order.

#### Column Compression

* When sequences of values in column are repetitive, they are a good candidate for compression.
    * E.g. *bitmap encoding*:
        * Encodes columns where there are only `n` distinct values with `n` bitmaps, one for each distinct value and one bit for each row.
        * The bit maps can then be run length encoded.
* Also allows for faster processing via taking advantage of CPU cyles and L1 cache.

#### Sort Order in Column Storage

* Don't *need* a sort order: can just keep in order each row is inserted.
* Columns must all be sorted together so the `k`th item from column belongs to the same row as the `k`th item from another column. So data must be sorted an entire row at a time.
    * Developer can choose which column to sort by (first, second, etc.). Which also allows for compression.
    * Example: sort by date, then product.
* Can also have different sort orders on different replicas and choose which one to query.

#### Writing to Column-Oriented Storage

* Column optimizations for reads make writes more difficult:
	* Can't update-in-place with compressed columns without rewriting all the column files.
* LSM-trees > B-trees for this with in-memory store then written to disk.

#### Aggregation: Data Cubes and Materialized Views

* *Materialized aggregates*: cache some common aggregations of data (e.g. `COUNT`, `SUM`, `AVG`, `MIN`, etc.).
    * Can use a *materialized view*: the result of some query that's actually cached to disk.
    * When data updates, a materialized view needs to update because it is a denormalized copy of data.
        * Make writes more expensive, which is why they're more common in data warehouses than OLTP databases.
    * *Data cube* or *OLAP cube*: store aggregate (e.g. `SUM`) of an attribute of all facts with some combiation of dimensions. See Figure 3-12 on pg. 102.


## Chapter 4 - Encoding and Evolution

Because of staged server rollouts and installed client apps, software needs to handle old/new data formats and schema changes (*backwards* and *forwards* compatability).

### Formats for Encoding Data

* The way data is represented in a program (as in-memory data structures) is different from when it is sent over the wire or written to a file.
* Transforming data from in-memory to a byte sequence is *encoding/serialization* and the reverse is *decoding/deserialization*.

#### JSON, XML, and Binary Variants

Downsides of textual formats:

* Ambiguity around interger, floating point numbers and precision.
* No support for binary strings.
* Schemas are only optional and many applications without schemas rely on hard-coded encoding/decoding rules.
* CSV is vague.
* JSON/XML require the field names themselves in the data, thus making them large (even when binary encoded).

#### Thrift and Protobufs

* Require a pre-defined schema.
* Each field has a type annotation, a field tag (name alias), and length information in addition to the data itself.

Personal note: `required` fields are considered harmful and were removed in proto3.

* *Schema evolution* is handled by field tags: old code can ignore new fields, and new code can always read old data as long as field tags are not changed. Never re-use a tag number.
* It's OK to change ints from 32->64 bits and `optional` to `required` fields.
* Thrift has lists, Proto just has `repeated` fields. OK to change `optional` to `repeated`.

Personal note: `message`s can be nested.

#### Avro

* Has two schema languages (IDL - human readable and JSON - machines).
* There are no field tags, so encoded data *must* be read with the matching schema.
* Instead, there is a *writer's* (built into app when encoding) and *reader's schema* (built into the app that's decoding data).
    * When reading data, Avro looks at the writer's and reader's schemas side-by-side and resolves differences. E.g. it's OK if fields are in different orders (they are matched by field names) or if there are missing/extra fields.
* To maintain compatibility, can only add or remove a field with a default value (must explicitly set fields as nullable).
* Three ways to include the writer's schema:
    * Large file with lots of records: include schema once at top of file.
    * Database with individually-written rows: include version number per-row and store version:schema in a table.
    * Sending records over network connection: send schema once at beginning of connection and use throughout connection lifetime.
    * *Not* having field numbers make it easier to use dynamically-generated schemas (e.g. dump contents of a db).

#### Merits of Schemas

* More compact since they eliminate field names.
* The schema itself is valuable documentation.
* Backwards and forwards compat.
* For statically-typed languages, generated code with type checking.

### Modes of Dataflow

#### Dataflow through Databases

* Writing to a database is encoding, reading from it is decoding. Writing to a database is passing a message to your future self.
* Databases are often written/read by older/newer versions of an application and must maintain backward and forward compat. Older code must be able to read records written by newer code, update the record, and maintain unknown fields.
* *Data outlives code* and migrations are expensive, usually it's preferable to add additional columns with default `null` values.
* Taking an archival snapshot of a DB (e.g. for backup or data warehousing) typically uses the most-recent version of the schema.

#### Dataflow through services: REST and RPC

* Applications are often split up as *client* (e.g. browser or mobile app) and *server*. The API exposed by the server is a *service*.
* A server can be a client to another server (e.g. a server to a database or another server). This is known as *microservices* or *SOA* (*service-oriented architecture*).
* Databases allow arbitrary queries to be run whereas services provide specific APIs with predefined business logic/encapsulation.

**Web Services**

Used for:

1. Client<->server.
1. Server<->server (within own company/datacenter), aka middleware.
1. Server<->server (between companies e.g. for data exchange).

**REST**

* Design philosophy building off of HTTP. Uses URLs for identifying resources and HTTP features for cache control, authentication, content type negotiation.
* Gaining in popularity. Less standards, but can use, e.g., Swagger to describe REST APIs and produce documentation.

**SOAP**

* XML-based protocol over HTTP but does not rely on HTTP features, instead re-builds them as part of the *web services framework* (*WS-\**).
* API is described using XML veriant called WSDL (Web Services Description Language) which enables code generation for local classes/methods for making API calls.
* WSDL and SOAP messages are not easily human-readable, so requires tooling support.
* Burdensome to integrate with, falling in popularity.

#### Problems with RPCs

* RPC *tries* to make making a network request the same as making a local function call, but there are fundamental differences:
    * Network requests are unpredictable (can fail) and you must anticipate and, e.g., retry.
    * A request could simply not return (e.g. timeout) and you may not know what happened.
    * If you retry, you may actually cause unintended consequences unless you build *idempotence*.
    * Network requests take a long time and have variable completion times.
    * All parameters to a network request must be encoded and sent over the wire.
    * Client and server may be implemented in different programming languages.

#### Current direction of RPCs

* Despite the above, RPC frameworks abound and are now more realistic about the differences between remote and local procedure calls.
* *Futures/promises* encapsulate async actions that may fail.
* *Streams* = a series of requests and responses.
* *Service discovery* = find IP/port of a service.
* RPCs for internal APIs, REST for public APIs.

**Data encoding and evolution for RPC**

* OK to assume servers will be deployed before clients, so OK to only hace backward compat.
* For public APIs, often need to version via URL, `Accept` header, or API key/admin dashboard because org has no control over client upgrades.

#### Message-Passing Dataflow

*Async mesesage-passing*

* Low latency like RPCs. Message is not sent via direct connection (like DB), but goes through an intermediary (*message broker/queue*).
* Advantages:
    * Queue acts as a buffer if recipient is unavailable/overloaded (improves reliability).
    * Can re-deliver messages to crashed processes, previnting loss.
    * No need to know IP/port of recipeint.
    * One message can be sent to many recipients.
    * Logically decouples sender/recipient.
* Message passing can only go one-way though.

**Message Brokers**

* E.g. Kafka.
* Process sends message to a `queue` or `topic` and *consumers*/*subscribers* to that queue/topic.
* Can be many producers/consumers per topic.
* Topics are one way, so to have two-way data flow, must reply to a queue that is subscribed to by the original sender.
* Choose your own encoding and data model. Backwards and forwards compat preferrable.

**Distributed Actor Frameworks**

* The *actor model* is a programming model for concurrency. *Actors* encapsulate logic and communicate by async sending messages. This can be scaled up to multiple nodes.
