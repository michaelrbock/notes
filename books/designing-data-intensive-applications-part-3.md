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


