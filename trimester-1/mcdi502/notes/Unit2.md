# Unit 2 - Data preprocessing and manipulation with Spark

## Objetives

Process and manipulate distributed data using Apache Spark, implementing RDDs, DataFrames, and Spark SQL queries to apply analytical techniques in big data environments. By the end of this unit, you will be able to perform transformation, integration, and optimization operations on massive datasets, demonstrating technical domain and an understanding of their application in real-world scenarios.


### Handling distributed data with Apache Spark

Work progressively with distributed processing tools, advancing from handling basic data structures to integrating, transforming, and preparing information in Big Data environments.

Spark operates under a Driver/Executor model: A central `Driver` program that orchestrates the work and multiple distrubuted `Executor` processes in a cluster that run tasks in parallel.
The role of the `Cluster Manager` (such as YARN or Kubernetes) was also introduced as the entity responsible for allocating physical resources to our application.

If the Driver schedules and the Executors execute, what do they operate on? How does Spark represent data in a way that allows for parallel, fast, and fault-tolerant processing? The answer to these questions lies at the very heart of Spark's first generation: the `Resilient Distributed Dataset (RDD)`.


#### Fundamentals of distributed processing with Resilient Distributed Datasets(RDDs)

The core innovation of Apache Spark, and what originally differentiated it from Hadoop MapReduce, is its abstraction for representing distributed data. This abstraction is the RDD.

An RDD is a fault-tolerant collection of elements that can be operated in parallel.
This definition, though concise, encapsulates the full power of Spark's programming model. Let's break it down in the context of what we already know:

* **Dataset (Collection of Elements)**: An RDD is an abstraction over a collection of objects. These objects can be simple, like lines in a text file (Strings), or complex, like customer records or sensor data. For the developer, it feels like working with a local collection (a list or an array), but at a massive scale.
  
* **Distributed**: This is where Spark's architecture comes into play. The data that makes up an RDD doesn't reside on a single machine. It's divided into logical fragments called partitions, and these partitions are distributed across the memory of the different Executors in the cluster. Spark handles this distribution completely transparently.
  
* **Resilient**: In a cluster with hundreds of nodes, failures aren't a possibility, but a certainty. "Resilience" is Spark's ability to automatically recover from the loss of a working node (and, therefore, the data partitions it hosted). Spark achieves this not through costly data replication, but through an ingenious mechanism called lineage.

Essentially, an RDD is a read-only, partitioned object that represents a dataset. It's important to note that an RDD doesn't always contain the physical data itself. Rather, it's a descriptor object that contains the "recipe" for calculating its partitions, either by reading them from an external source (such as HDFS or S3) or by applying a series of transformations to another RDD.


**Why were RDDs created? The MapReduce problem**

Hadoop MapReduce pioneered large scale distributed processing, allowing organizations like Google to process petabytes of data. However, its excution model had funtamental limitations.

A MapReduce job consists of a Map phase and a Reduce phase. Crucially, the intermediate results of the Map phase must be written to the Hadoop Distributed File System (HDFS) before the Reduce phase can begin. This approach presents two serious problems:

* I/O Latency: Disk read/write access is orders of magnitude slower than RAM. Because MapReduce constantly writes intermediate results back to disk, multi-step workflows accumulate heavy I/O overhead, leading to major processing delays.

* Ineffienciency in iterative algorithms: Algorithms like Machine Learning (K-Means, Logistic Regression) and graph analysis (PageRank) process the same dataset repeatedly. MapReduce forces a full disk read-and-write cycle for every single iteration (e.g., 10 iterations = 10 full disk I/O passes), making repeated tasks painfully slow.
  
* Lack of interactivity: MapReduce is poorly suited for interactive data exploration or fast querying. Each query runs as an isolated job that must flush its results to disk before the next job can begin, eliminating any chance of rapid feedback loops.

The RDDs were specifically designed to solve these problems by introducing two revolutionary concepts: in memory processing and intelligent persistence.

**Spark vs MapReduce**

Here is the Markdown table translated into English:

| Aspect | Hadoop MapReduce | Apache Spark with RDDs |
| --- | --- | --- |
| **Intermediate Storage** | HDFS (disk) | RAM memory |
| **Speed (typical)** | 100x slower | 100x faster |
| **Iterative Algorithms** | Very inefficient | Highly efficient |
| **Interactive Analysis** | Impractical | Excellent |
| **Programming Model** | Rigid (Map/Reduce) | Flexible (RDD API) |
| **Fault Tolerance** | Data replication | Lineage (recompute) |
| **Failure Recovery** | Slow (rewrite data) | Fast (recompute partition) |

**The five fundamental properties of an RDD**

1. List of partitions (`partitions()`):
   * The logical division of the dataset and the atomic unit of parallelism.
   * Defines how the data is split across the cluster (e.g 1 GB file on HDFS divided into eith 128 MB parititions).
2. Compute Function (`iterator(split, context)`):
   * The execution logic used to compute each partition.
   * Returns an iterator over a partition's records, this code is serialized and sent directly to a cluster `Executors`
3. List of Dependencies (`dependencies()`):
   * The RDD's lineage, tracking its parent RDDs.
   * Classifies transformations as either Narrow (1 to 1 partition mapping like `map()`, no shuffle) or Wide (1 to many mapping like `groupByKey()`, requires a network shuffle).
4. Partitioner (`partitiones`)(optional):
   * Specifies how keys are distributed across partitions in Key-Value RDDs (e.g via HashPartitioner)
   * Optimizes operations like `join()` by preventing expensive network shuffles if both RDDs share the same partitioner.
5. Preferred Locations (`preferredLocations(split)`)(optional):
   * Identifies which physical cluster nodes hold the underlying ata blocks (e.g HDFS block locations)
   * Drives data locality by allowing the scheduler to run tasks on the nodes where the data already resides, minimizing network traffic.

Understanding this anatomy makes it clear that an RDD is not simply a data container. It is an immutable, distributed execution plan. It is a recipe that tells Spark how to produce a dataset, how to do so in parallel, and how to rebuild it if something goes wrong.