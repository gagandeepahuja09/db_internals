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

**Durability in Memory-based Stores**
* Before the operation can be considered complete, its results have to be written in a sequential log file.
* To avoid replaying complete log contents during startup or after a crash, in-memory stores maintain a *backup copy*.
* The backup copy is maintained as a *sorted disk-based structure*, and modifications to this structure are often async (decoupled from client requests) and applied in *batches to reduce the no. of I/O operations*.

* During recovery, database contents can be restored from the backup and logs.
* Backup holds a database snapshot for a specific point in time and log contents up to this point can be discarded. This process is called *checkpointing*.

* *Note*: In memory-DB are not the equivalent of on-disk DB with a huge page cache. Even though pages are cached in memory, *serialization format and data layout incur additional overhead* and do not give the same degree of optimizations as in-memory stores.

* Disk-based databases use specialized storage structures, optimized for disk access.
* Random memory access is significantly faster than random disk access.
* Disk-based storage structures often have a form of wide and short trees.
* Memory-based implementations can choose from a larger pool of data structures and choose optimizations that would otherwise be impossible or difficult to implement on disk.
* Handling variable-size data on disk requires special attention, while in memory it's often a matter of refrencing value with a pointer.

**Column v/s Row-Oriented DBMS**

**Row-oriented layout**
* Because data on a persistent medium such as a disk is typically accessed block-wise (block is the minimal unit of disk access), a single block will contain data for all columns. Great for use cases where would like to access an entire user record.

**Column-oriented layout** 
* To reconstruct data tuples for joins, filtering, multirow aggregations, we need to preserve some metadata on the column level to identify which data points from other columns it is associated with.
* If we do this explicitly, each value will have to hold a key, which introduces duplication and increases the amount of data.
* Some column stores use implicit identifiers (virtual IDs) instead and use the position of the value (offset) to map it back to the related values.

**Distinctions and Optimizations**
* Reading multiple values for the same column in one run significantly improves cache utilization and computational efficiency.
* Storing values that have the same data type together offers a better compression ratio.

**Wide Column Stores**
* Eg. BigTable, HBase. Data is represented as multi-dimensional map.
* Columns are grouped into column families (usually storing data of same type).
* Inside each column, data is stored row-wise.
* This layout is best for storing data retrieved by a key or a sequence of keys.
* Each row is indexed by its row key.
* Related columns are grouped together in column-families which are stored on disk separately. Data belonging to the same key is stored together in a column family.
* Each column in a column family is identified by the column key, which is a combination of the column family name and qualifier (html, abc.com, zyx.com)
* WebTable example

{
    "com.cnn.www": {
        "contents": {
            t1: html: "<html>..."
            t2: html: "<html>..."
        }
        "anchor": {
            t8: html: "<html>..."
            t9: html: "<html>..."
        }
    }
    "com.example.www": {
        "contents": {
            t7: html: "<html>..."
        }
        "anchor": {}
    }
}

**Data Files And Index Files**
* How does file organization improve efficiency?

* *Storage efficiency*:
    * Files are organized in a way that *minimizes storage overhead per stored data record*.
* *Access efficiency*:
    * Records can be *located in the smallest possible no. of steps*.
* *Update efficiency*:
    * Record updates are performed in such a way that *minimizes the no. of changes on disk*.

* Each table is usually represented as a separate file.
* Each record can be looked up using a search key.
* To locate a record, DBs use indexes: avoiding entire table scan.

* Files are partitioned into pages, having size of single or multiple disk blocks.
* Pages can organized as sequence of blocks or as slotted pages.

* New record (insertions) and updates to the existing records are represented by key/value pair.
* Most newer DBs use garbage collection for deletion. 
    * Reads the pages, writes the live records to new place and discards the old ones.

**Data files**
* Data files can be organized as: 
    * Index-organized tables(IOT)
    * Heap files
    * Hash files

* *Heap Files*:
    * Records are not required to follow any order. Most of the time it is write order.
    * No additional work or file reorganization is required when new pages are appended.
    * The require additional index structures pointing to the location where data records are stored, to make them searchable.

* *Hashed Files*:
    * Records are stored in buckets.
    * Hash value of the key determines which bucket a record belongs to.
    * Records in a bucket can be sorted in an append order or by key.

* *Index organized tables*:
    * Storing data records in the index itself. This allows us to reduce the no. of disk seeks by atleast one, since after traversing the index and locating the searched key, we don't have to address a separate file to find the associated data record.
    * Records are sorted by key, which helps speed up range queries as we can sequentially scan the contents.

**Index files**
* Secondary index can point directly to the data record or simply store its primary key.
* While primary index files hold a unique entry per search key, secondary indexes may hold several entries per search key.
* With a clustered index, the rows are stored in the same order as the index. Due to this, there can only be one clustered index, ie the primary index. Hence, for range queries and various other options, it could be a better option to change the primary index if possible, rather than creating a secondary index.
* Index-organized tables store information in index order and are clustered by definition.

**Primary Index as an Indirection**
* There are varying opinions in the db community on whether data records should be referenced directly or via the primary key index.

* By referencing the data directly, we can reduce the no. of disk seeks but have to pay a cost of updating the pointers whenever the record is updated.

* For write-heavy workloads, using primary key as an indirection can be a good option as we need not update the pointers to secondary indexes. Eg. done by InnoDB.

* Hybrid approach: We store both data files and primary keys. If the position has changed, we pay the extra cost of going through primary key and updating the index file after finding the new offset.

**Buffering, Immutability and Ordering**
* Storage structures have the above 3 common variables.

* *Buffering*
    * This defines whether the storage structure chooses to collect a certain amount of data in more before putting it on disk. ( we are taking about Avoidable buffering )
    * Unavoidable buffering: Every on-disk structure has to use buffering to some degree, since the smallest unit of data transfer to and from disk is a block and it is desirable to write full blocks.
    * Optimization examples:
        * Lazy B-trees: Adding in-memory buffers to B-tree nodes to ammortize I/O costs.
        * Two-component LSM trees.

* *Immutability*
    * Common immutable ways:
        * Append-only: Modifications are appended to the end of the file.
        * Copy-on-write: Modified page, holding the updated version of the record, is written to the new location in the file instead of its original location.
    * Often, the distinction b/w LSM, B-tree is though as immutability, but there are structures like B-w trees whihc are inspired by B-trees and are immutable.

* *Ordering*
    * Ordering defines whether or not we can effeciently scan a range of records, not only locate the individual data records.
    * Insertion order --> write time optimization.