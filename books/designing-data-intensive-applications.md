# Designing Data-Intensive Applications

[Goodreads](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications)

-

### Chapter 1 - Reliable, Scalable, and Maitainable Data Systems

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

### Chapter 2 - Data Models and Query Languages

