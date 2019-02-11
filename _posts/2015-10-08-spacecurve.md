---
layout: post
title: "SpaceCurve: An Unique And Fast Geospatial Database"
---

The design of SpaceCurve has been the object of speculation for many years. SpaceCurve was created to provide a real-time, continuously updated, perpetually logged data view of everything that happens in the physical world for analysis at extreme scales. The term "database" is used loosely.

In principle, a single dense rack of commodity Linux servers can do the following simultaneously:

* Continuously insert a billion GeoJSON documents per minute through disk. This includes parsing JSON and indexing polygons in 3-space.
* Execute real-time queries that can seamlessly blend new and old data. Queries are automatically and transparently parallelized across the cluster.
* Fuse dozens of live data sources with spatiotemporal joins to generate real-time context for billions of people, places, and things at global scales. 

It is, as one government user put it, "ungodly ridiculously fast". The above gives a flavor of what it can do and why it exists, but this post is about how it does it.

SpaceCurve is a prototype of a new class of database architecture. It fuses two novel computer science foundations that make it optimal for workloads that mix extremely high concurrent write rates with low-latency parallel queries. First, it is a **CPU sharding software architecture**, which radically improves operation throughput rates relative to multithreaded software architectures on modern hardware. Second, the database is **built on discrete topology** internals, not ordered sets, which is more efficient for massively parallelizing most database operations, notably geospatial and join operations. The combined architecture is state-of-the-art for both scale-up and scale-out databases. Graph, document, or relational databases could be designed with similar characteristics.

### CPU Sharding Software Architecture

CPU sharding architectures originated in high-performance computing (HPC) as an answer to the problem of proliferating core counts in CPUs. A modern server is a large distributed system in every sense. Shared resources on a single server therefore require coordination to ensure consistency. As with other distributed systems, even coordination protocols in hardware (e.g. lock-free algorithms) become expensive as the number of cores grows. Multithreaded software architectures rely on ubiquitous coordination and data movement across server silicon; CPU sharding architectures attempt to virtually eliminate it.

CPU sharding brings “shared nothing” down into the silicon of a single server. It is an atypically effective macro-optimization and can improve operation throughput upwards of 10x versus conventional multithreading. In these architectures, every core has a single process locked to it and CPU-local RAM that it shares with no other process. This can be extended to I/O to the extent possible, giving each process direct access to dedicated hardware queues for networking and bypassing the kernel for disk operations. Every process shares its silicon with as few other processes as the hardware and operating system allow. There is only marginally more communication between cores on the same CPU than there is between cores on different servers. [ScyllaDB](http://www.scylladb.com), recently announced, is a rare open source example of a CPU sharding design.

This architecture trades one type of complexity for another. It virtually eliminates thread concurrency overhead and complexity while greatly reducing average latencies across every part of the code base during execution. However, these designs also require every process to be a sophisticated scheduling engine at its core, a something traditionally left to operating systems. 

SpaceCurve’s database kernel is a conventional CPU sharding architecture, directly implementing its own I/O and execution schedulers in user space. It also has some additional elements that extend the performance beyond what is implied by CPU sharding:

* Queries are dynamically compiled with LLVM
* Discrete topology internals largely obviate secondary indexing
* Thousands of shards per process with continuous rebalancing under load
* Optimized C++11 with few external dependencies, about 250,000 lines.

A philosophy of implementation is that acceptable bottlenecks are network bandwidth, usually 10 GbE, and memory bandwidth. People are frequently surprised that disk I/O is not on that list, because of the implication that the software should be at least as fast as in-memory databases given otherwise identical hardware. Modern disk storage has sufficient bandwidth for most workloads such that being in-memory solves a performance problem only to the extent that having no I/O scheduler is faster than a poor I/O scheduler.

### Parallel Computing With Discrete Topology

Databases built on manipulating ordered sets of records cannot scale-out operations on geospatial data models even in theory. The reason is simple: the underlying data types have no order and are not set-wise shardable, eliminating almost every technique a typical database has for efficiently representing and manipulating data models at scale. Computational representation of data models that do not meaningfully map to ordered sets has been studied with little success since the 1960s, but this had few practical implications for the average developer until recently. Outside of HPC codes, geospatial, and complex event processing, you were unlikely to see an application where it mattered.

In 2006, I started working on the design of algorithms for a distributed database based purely on interval data types, which in theory *could* express geospatial data models efficiently, [having learned the hard way that solving these problems was unavoidable](http://www.jandrewrogers.com/2015/03/02/geospatial-databases-are-hard/). The computer science literature had little to offer but I found hints of how one might attack these problems in the information theory and topology literature. A fully generalized solution produces a deep stack of novel computer science built on top of discrete topologies instead of ordered sets. Proper implementations have exotic characteristics:

* The only primitive types are hyper-rectangles of infinite volume
* Arbitrary compositions of primitives are computationally homomorphic
* Data model representation is adaptive, distributable, and universal
* Most algorithms built on topology manipulation are inherently parallelizable

Surprisingly, good implementations are compact, efficient, and readable, losing nothing to ordered set algorithms except familiarity. However, even if you can see what an implementation does, the *why* is rarely obvious to the uninitiated.

SpaceCurve’s primary internal data structures and algorithms are all based on parallelized discrete topology, simplified to reflect a geospatially optimized use case. When you run a SQL query, it is immediately translated into this. Discrete topology implementations have several advantages for databases generally:

* **A record and a constraint are literally the same thing.** They can be stored and processed the same way. By implication, efficient stream processing of the insertion pipeline is a trivial extension of ordinary record selection, both of which are content-addressable operations.
* **Relationships between arbitrary constraints are content-addressable.** The computational cost is similar to hashing while preserving selectivity across a *much* richer set of relationships. By implication, secondary indexing is not that useful.
* **Every data model is represented by the same "indexing" structure.**  Social graph and geospatial polygon data models use the same adaptive organization and access method under the hood. The operations that can be directly applied are not dependent different organizations.
* **Ad hoc joins are efficiently and massively parallelizable.** The expressiveness of join-like traversals within and between data models is a unique characteristic of topological implementations. SpaceCurve's algorithms were originally prototyped as a parallel graph analysis engine. 

There is one significant drawback that should not be understated. Algorithm design using topology manipulation can be enormously challenging to reason about. You are often taking a conceptually simple algorithm, like a nested loop or hash join, and replacing it with a much more efficient algorithm involving the non-trivial manipulation of complex high-dimensionality constraint spaces that effect the same result. Routinely reasoning about complex object relationships in greater than three dimensions, and constructing correct parallel algorithms that exploit them, becomes easier but never easy.

### Related Trivia

The platform contains a state-of-the-art non-Euclidean geometry engine designed from scratch, with particular focus on correctness and precision. It is used to reason about relationships in its internal 3-space model of the physical world. The extent to which other popular geospatial platforms regularly produce different results due to defects in their geometry engines is astonishing. 

Literature on CPU sharding *per se* is scarce but the technical skills to implement it are not and many software systems have been built this way. I expect to see more software designed this way in the future, the benefits are hard to ignore.  

There is virtually no literature on practical representations of topological spaces, never mind parallel algorithms using those representations. A thorough exposition of both the theory and practice is on the order of a few hundred pages of dense technical literature that no one has had time to write, despite multiple implementations. Watch this space. 

Internally, we call “CPU sharding + discrete topology” database architectures the Madrid architecture. I wrote an early definition of the draft concept in a hotel room in Madrid and the name stuck. 


