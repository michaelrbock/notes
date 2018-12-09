# Designing Data-Intensive Applications

[Goodreads](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications)

-

## Chapter 1 - Reliable, Scalable, and Maitainable Data Systems

####*Reliability*: The system should work correctly, even in the case of hardware/software/human error.

* Includes correctness, edge case handling, speed, security.
* Types of faults (that could lead to *failures*):
  * Hardware (usually unrelated, but common).
  * Software (often cascading, can lie dormant until some unusal circumstance).
  * Human; Mitigations:
	  * Well-abstracted APIs
	  * Sandboxes
	  * Testing (unit, itegration, manual)
	  * Allow rollbacks
	  * Monitoring
	  * Good management and training

#### *Scalability*: As the system grows, there should be reasonable ways to deal with that growth.

* No: "X is scalable", "Y doesn't scale".
* Yes: "If the system grows in a particular way, wat are our options for coping?".
* *Load parameters*: e.g. requests/second, cache hit rate, read/write ratio, average/peak.

Example: Twitter Timeline, two approaches:

1. On timeline load, get tweets from all users the current user follows, merge them and sort by time.
1. Maintain a cache for each users' timeline. When someone tweets, insert that into each of their followers' timeline caches.

* Two questions to assess performance:
  1. When you increase a load parameter and keep system resources the same, how does that affect performance?
  1. How much would you need to increase resources to keep performance the same if you increased a load parameter?
* Describing performance: e.g. throughput for a batch system or response times (use percenntiles).
* *Head of line bocking*: slow requests block the server from responding to subsequent requests.
* *Tail latency amplification*: the slowest backend call (even when made in parallel) will determine end user response time.
* Types of scaling:
  * Vertical
  * Horizontal
  * Shared-nothing: distributing load across multiple machines
  * Elastic: automatically increases resources under load

#### *Maintainability*: People in the future should be able to work on the system *productively*.

* Operability: allow operations to keep system running smootly.
  * Monitoring, security, configuration changes, maintence, documentation.
* Simplicity: easy for new engineers to understand the system.
  * Remove *accidental* complexity. Use good *abstraction*.
* Evolvability/extensibility/evolvability

## Chapter 2 - Data Models and Query Languages

#### Relational Models vs. Document Model

Reasons for NoSQL adoption:

* Need for greater scalability (very large datasets or high write throughput)
* F/OSS > commercial
* Specialized query operations
* More dynamic/expressive data model

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

This is called data *normalization* and relies on *many-to-one* relationships.

Cons:

* If the document DB itself does not support joins, must emulate in application code by making multiple queries.
* Data tinds to become more interconnected as features are added, which often have *many-to-many* relationships.

#### Relational vs. Document Databases Today

* Use a document model if your application data has a document-like strucutre (i.e. a tree of one-to-many relationships, where typically the entire tree is loaded at once).
    * This avoid *shredding* in relational model which splits a document into multiple tables.
    * Avoid *too*-deeply nesting data.
    * Be careful of poor support for joins. 
        * If your application has many-to-many relationships, the document model is less appealing.
        * De-normalizing helps, but application needs to keep denormalized data consistent.
        * Joins can be emuluated in application code, but is usually slower than DB joins.

* Document databases are usually *schema-on-read* (implicit schema, interpreted whhen read), vs. *schema-on-write* in relational DBs.
* Schema changes in dociument model usually involve `if/else` statements in code vs. `ALTER TABLE` migration in relational model.

**Data Locality**

* Documents are stored continuously and have performance advantage of *storage locality*.
* Only applies if you need much of the document at the same time: wasteful if you only need a part of it at a time or need to do writes (the whole document needs to be re-written).

### Query Languages

#### Declarative Languages

* SQL is a *declarative* language which specifies thhe pattern of data you want (as opposed to a *imperative* programming language where you specify how to achieve goal). 
* Declarative languages hides implementation details which allows performance improvements in database engine. 
* Declarative languages lead to parallel execution.
* CSS is similarly declarative.

#### MapReduce Querying

* MonogoDB implements a MapReduce pattern which mixes imperative (*pure* `map` and `reduce` functions) with a declarative query language.
* Distributed SQL queries *can* be implemented as a MapReduce pipeline.

### Graph-Like Data Models

* Useful for data with common many-to-many relationships, e.g.
    * Social graphs, Web pages with links, Road or rail networks.
* Verticies do not have to be *homogeneous*: you can represent many different types of objects in a single graph.

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

For Neo4j: a declarative language for modeling and querying graph datastore.

#### Graph Queries in SQL

Can use `WITH RECURSIVE` SQL syntax, but it's much more complicated than Cypher's declarative statement.

#### Triple Stores and SPARQL

All information is stored in three-part statements: (*subject*, *predicate*, *object*), e.g. `(Jim, likes, bananas)`.

Can store properties, e.g. `(lucy, age, 33)` or relationships, e.g. `(lucy, marriedTo, bob)`.

**Semantic Web**

A project to represent web sites in a machine-readable format in addition to human-readable. Uses RDF: Resource Description Framework.

## Chapter 3 - Storage and Retrieval

