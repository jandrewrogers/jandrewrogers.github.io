---
layout: post
title: Why Are Geospatial Databases So Hard To Build?
---


In 2005, I tried to build my first large-scale geospatial database. PostGIS was unable to keep up with even modest volumes of fast-moving sensor data, so I decided to apply my knowledge of scalable database engines to the problem of analyzing geospatial data. I had minimal experience with geospatial database design, [but how hard could it be](http://en.wikipedia.org/wiki/Dunning%E2%80%93Kruger_effect)? As I slowly discovered, the computer science required to build a competent, scalable geospatial database did not even exist.

Most software engineers assume today, as I did a decade ago, that the design of geospatial databases is a straightforward task. While not always obvious, nothing could be further from the truth. The typical methods for representing dynamic geospatial data, from R-trees to hyperdimensional hashing to space-filling curves were all invented in the 1980s or earlier. If you could actually build scalable geospatial databases that way, people would have. 

Geospatial databases, at a fundamental computer science level, are unrelated to more conventional databases. The surface similarities hide myriad design challenges that are specific to spatial data models. All of the design challenges below are lessons I learned the hard way, manifested as critical defects in real-world applications. You can think of it as a checklist for "is my geospatial database going to fail me at an inconvenient moment". 


### Computer Science Does Not Understand Interval Data Types


Algorithms in computer science, with rare exception, leverage properties unique to one-dimensional scalar data models. In other words, data types you can abstractly represent as an integer. Even when scalar data types are multidimensional, [they can often be mapped to one dimension](http://en.wikipedia.org/wiki/Space-filling_curve). The majority of data people care about can be represented with scalar types, so it works well. 

If your data model is inherently non-scalar, you enter an algorithm wasteland in the computer science literature. Paths, vectors, polygons, and other elementary aggregations of scalar coordinates used in spatial analysis are non-scalar data types. Computational relationships are topological instead of graph-like.

Spatial data types, among a few other common data types, are _interval data types_. An interval data type cannot be represented with less than two scalar values of arbitrary dimensionality, like the boundary of a hyper-rectangle. These differ from scalar types in two important ways: sets have no meaningful linearization and intersection relationships are not equivalent to equality relationships. The algorithms that do exist in literature for interval data are poor.

* **There are no dynamic sharding algorithms that produce uniform distributions of interval data**. In fact, you can show that a general partitioning function that uniformly distributes _n_-dimensional interval data in _n_-dimensional space does not even exist. Hash and range partitioning, which in this context is essentially a distributed Quad-Tree, are hammers for a problem that is not a nail. 

* **Interval indexing algorithms in literature are either not scalable or not general**. Ironically, the literature describes hundreds of algorithms. The R-Tree family of algorithms, the traditional choice for small data sets, cannot scale even in theory. The Quad-Tree family of algorithms are pathological for interval data; R-Trees were invented to replace Quad-Trees for this reason. Sophisticated variants of grid-file indexing ("tiling") can scale for static interval data but fail for dynamic data and most software engineers use even more naive variants in practice.

* **Interval data sets are not sortable and have no exploitable pseudo-order**. The assumption that data has exploitable order is so pervasive in computer science that many design patterns either have no benefits or are incorrect when applied to interval data. For example, what is the best storage format for an analytical geospatial database? The answer is not obvious. Many storage formats, such as those in columnar databases, make tradeoffs predicated on data being sortable.


### Database Engines Cannot Handle Real-Time Geospatial


Most geospatial databases were built for creating maps. As in, geospatial data models that can be rendered as image tiles or paper products. Mapping databases evolved in an environment where the data sets were small, rarely changed, and production of a finished output could take days to complete.

Modern spatial applications are increasingly built around real-time spatiotemporal analysis and contextualization of data from IoT, sensor networks, and mobile platforms. These workloads look nothing like making maps. In fact, these workloads are unlike any workload studied in database literature. And therein lies the challenge: any database engine optimized for a modern geospatial workload must be designed from first principles. There are no “shoulders of giants” in database engines to stand on because these workloads were never on their radar.

* **Many spatial data sources continuously generate data at extremely high rates**. The Twitter firehose is barely a trickle compared to many high-value data sources that must be spatially organized. Not only do you need to parse, index, and store complex spatial data at rates of petabytes per day, but this must be concurrent with low-latency queries against incoming and historical data that cannot be summarized.  In-memory databases are useless here; the volume of online data is too large. While technically feasible, few databases were designed for this workload.

* **Traditional disk storage and I/O scheduler designs pervasively assume a trivial ordered log or tree structure**. This reflects the typical organization of scalar data models. What is the optimal storage layout for interval data models with irregular multidimensional locality characteristics and no order except temporal? How do you design an I/O scheduler that can intelligently schedule operations in a way that reflects those relationships? Not answering this question results in an integer factor loss in effective storage bandwidth.

* **Geospatial workloads tend to be highly skewed and constantly shifting over the data model in unpredictable ways**. This creates the unique characteristic that you can’t mitigate hotspots with the traditional approach commonly used in time-series databases of making *a priori* assumptions about workload distribution and hardcoding it into the software. Needing to adaptively mitigate hotspots with unpredictable and severe skew is a novel requirement.


### Correct, Fast Computational Geometry Is Really Hard


Geospatial operations are inherently built around computational geometry primitives. Anyone can Google a description of the [Vincenty algorithm](http://en.wikipedia.org/wiki/Vincenty%27s_formulae) but implementing non-Euclidean operators that are correct, precise, and fast will likely be beyond their ken (and mine). Existing implementations were designed for cartographic use cases, where performance and precision are not as important as they are for analytics.

Geospatial analytics requires receiving trustworthy answers in a timeframe that matters. If you are modeling a global supply chain, whether corn or coal, a 2% error due to low-quality computational geometry has billion  dollar ramifications. Performance is often so poor that leading commercial vendors tout analytic results that are delivered in weeks for data sets that fit in memory. That may be good enough for making maps but it is nigh useless for modern sensor analysis applications.

* **The physical world is non-Euclidean**. The curvature of our Earth's gravity field is approximately 8 centimeters per kilometer. This produces significant horizon effects even within a single large building. The simplest geometric surface that still applies to most calculations is a geodetic ellipsoid. The computationally simpler approximations used under the hood by many popular platforms work for maps but not for spatial analytics. 

* **Geospatial analytics requires the ability to do polygon intersections quickly**. Polygon intersection algorithms, like relational joins, are quadratic in nature. Comparing polygons with thousands of vertices, which are common in some industries, may require a million geodetic comparisons to evaluate one record. Multiply that by billions of records and your query may not complete this month. This often shows up as inadvertent denial-of-service attacks in poorly designed geospatial databases.

* **Computational geometry has to be precise if you care about analysis**. For trivial algebraic geometry it is not difficult to guarantee the error bounds on floating point computations. Most programmers do not realize that the transcendental functions implemented in their CPUs exhibit significant losses of precision in a variety of edge cases that must be accounted for such that the native primitives cannot be trusted for calculations that matter and certainly not at scales where the range of those primitives will be thoroughly tested.

### Difficult, But Not Impossible

Any discussion of how to solve one of these problems is a long blog post on its own but suffice it to say for now that these are all solvable with sufficiently clever computer science and competent implementation. However, these problems are not trivial and some have solutions that are not in the public computer science literature.

With the emergence of mobile, IoT, and sensor analytics as high-value markets, every database platform is scrambling to "enable" spatial analytics. It is extremely difficult to add adequate support for high-performance spatial analytics to databases that were not purpose-built for that use case. Many database companies have tried and failed. 

From a database architecture standpoint, the root of the problem is that the internals of a database engine designed for traditional "text and numbers" data models, which is virtually all of them, has very little in common with a database engine designed for real-time spatial data models. No amount of software duct tape will get around this gross impedance mismatch. 

