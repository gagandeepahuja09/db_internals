* Neither focused on relational nor non-relational.
* Focus is on concepts used in all database management systems with a focus on storage engine and components responsible for distribution.
* Part 1(storage), Part 2(distribution)

**Part1: Storage Engine**
* Databases are modular systems and consist of many parts:
    1. A *transport layer* accepting requests.
    2. A *query processor* determining the most efficient way to run queries.
    3. An *execution engine* carrying out of the operations.
    4. A *storage engine*.
* DBMS are applications built on top of storage engines, offering a schema, a query language, indexing, transactions and many other useful features.
* This allows pluggability. MySQL can switch b/w storage engines InnoDB, MyISAM, RocksDB. MongoDB allows switching b/w WiredTiger, In-Memory.

**Comparing Databases**
* Comparing databases on the basis of their components(eg. storage engine, how data is replicated, shared and distributed)/rank/impl. language can lead to invalid and premature conclusions.
* *Every comparison should start by clearly defining the goal*, because even the slightest bias may completely invalidate the entire investigation.
* We should *stimulate the workload* against different databade systems, *measure the performance metrics that are important to us and compare results*.
* Simulating real-world workloads not only helps us understand how the DB performs, it also helps us learn how to operate, debug, and find out how friendly and helpful the community is.

* We should understand the use-case in great detail and define the current and anticipated variables, such as:
    1. Schema and record sizes.
    2. Number of clients.
    3. Types of queries and access patterns.
    4. Rates of the read and write queries.
    5. Expected changes in any of these variables.

* Knowing these variables, we can answer the following questions: 
    1. Does the DB support the required queries?
    2. Is it able to handle the amount of data we're planning to store?
    3. How many read and write requests can a single node handle?
    4. How many nodes should the system have?
    5. How do we expand the cluster given the expected growth rate?
    6. What is the maintainence process?

* Having these questions answered, we can construct a test cluster and simulate our workloads.
* It there is no standard stress tool to generate realistic randomized workloads in the DB ecosystem, it might be a red flag. Else we can use some general purpose tool or implement one from scratch.

* YCSB (Yahoo Cloud Serving Benchmark).
    * It offers a framework and common set of workloads that can be applied to different data stores.
    * Just like any other generic tool, we should use it with caution, since it's easy to make wrong conclusions.

* To make a fair comparison, it's important to understand the real-world conditions under which the DB has to perform and tailor benchmarks accordingly.

* Benchmarking is not just limited to DB comparison. It can be used for defining SLA, capacity planning, understanding system requirements, etc.

* We should have an update strategy. Testing before rolling out a new version is critical.

**Understanding Trade-Offs**

* Some of the challenges while designing a storage engine:
    * Designing the physical data layout and organizing pointers.
    * Deciding on the serialization format.
    * How data is going to be garbage collected.
    * How the storage engine fits into the semantics of the database system as a whole.
    * Figuring out how to make it work in a concurrent environment.
    * Making sure we never loose any data under any circumstances.

* Many of the decisions have trade-off eg. storing the data as it is or in a sorted manner.

**DBMS Architecture**

* *Transport Subsystem*
    * Client requests(application layer) arrive through this.
    * It is also responsible for communication with other nodes in the database cluster.

* *Query Processor*
    * Parses, interprets and validates the query.
    * Later access control checks are also performed as they can only be done after a query is interpreted.
    * The parsed query is passed to the query optimizer.
        * It first eliminates the impossible and redundant parts of the query.
        * Then attempts to find the most efficient way to execute it based on *internal statistics*
            (index cardinality, approximate intersection size, etc.) and *data placement*
            (which nodes in the cluster hold the data and the cost associated with its transfer).
        * The optimizer handles both *relational operations* required for query resolution, usually represented as dependency tree and *optimizations* such as index ordering, cardinality estimation, and choosing access methods.
    * The query is usually represented in the form of an execution plan (or query plan): a sequence of operations that have to be carried out for it results to be considered complete.
    * Since the same query can be satisfied using different execution plans that can vary in efficiency, the optimizer picks the best available plan.

* *Execution Engine*
    * It handles the execution plan by collecting the results of the execution of local and remote operations.
    * Remote execution can involve writing and reading data to and from other nodes in the cluster, and replication.

* *Storage Engine*
    * Local queries (coming directly from clients or from other nodes) are executed by the storage engine. Components:
    
    * *Transaction Manager*
        * It schedules transactions and ensures they cannot leave database in a logically inconsistent state.

    * *Lock Manager*
        * It locks on the database objects for the running transactions, ensuring that concurrent operations do not violate physical data integrity.

    * *Access methods (storage structures)*
        * These manage access and organizing data on disk.
        * Includes heap files and storage structures such as B-trees or LSM trees.

    * *Buffer Manager*
        * It caches data pages in memory.

    * *Recovery Manager*
        * It maintains the operation log and restoring the system state in case of a failure.

    * Together, transaction and lock manager are responsible for concurrency control. They ensure logical and physical data integrity while ensuring that concurrent operations are executed as efficiently as possible.

**Memory v/s Disk-Based DBMS**
* In-memory DBMS store data primarily in memory and use the disk for recovery and logging.
* Disk-based DBMS hold most of the data on disk and use memory for caching disk contents or as a temporary storage.
* Main memory database differ from disk-based not only in primary storage, but also in which data structures, organization, optimization techniques they use.
* Programming for main memory is significantly simpler than doing for the disk. *Operationg systems abstract memory management* and allow us to think in terms of allocating and freeing arbitrarily sized memory chunks.
* On disk we have to manage *data references, serialization formats, freed memory, and fragmentation manually*.

* Main limiting factors on the growth of in-memory databases are RAM volatility (or lack of durability) & costs. Since RAM contents are not persistent, software errors, crashes, hardware failures, and power outages can result in data loss.
* The situation is likely to change with NVM (non-volatile memory). 