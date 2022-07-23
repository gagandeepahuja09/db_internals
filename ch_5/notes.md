**Transaction Processing And Recovery**
* Bottom-up approach. First learned about storage structures.
* How higher-level components responsible for *buffer management, lock management, and recovery* which are the prerequisites for understanding database transactions.

* *Transaction*: Indivisible logical unit of work in a DBMS, allowing us to represent multiple operations as a single step.

* *Atomicity*: 
    * Transaction steps are indivisible, which means that either all of the steps associated with a transaction execute or none of them do.
    * Each transaction can either commit or abort. After abort, a transaction can be retried.

* *Consistency*:
    * It is an application specific guarantee.
    * A transaction should only bring the DB from one valid state to another valid state, maintaining all DB variants (such as constraints, refrential integrity and others).
    * It is the most weakly defined property because it is the only property that is controlled by the user and not the DB.

* *Isolation*:
    * Multiple concurrently executing transactions should be able to run without interference as if there were no other transactions executing at the same time.
    * Isolation defines *when* the changes to the DB state may become visible, and *what* changes may become visible to the concurrent transactions.
    * Many DBs use default isolation levels that are weaker than the given definition for performance reasons.
    * Depending on the methods and approaches used for concurrency control, changes made by a transaction may or may not be visible to other transactions.

* *Durability*:
    * Once a transaction has been committed, all DB state modifications have to be persisted on disk and be able to survive power outages, system failures and crashes.

* Implementing transactions in a DBMS, in addition to a storage that organizes and persists data on disk requires several components to work together.

* *Transaction Manager*
    * It *coordinates, schedules and tracks* transactions and their individual steps.

* *Lock Manager*
    * Whenever a lock is requested, the lock manager checks if it is already held by any other transaction in shared or exclusive mode.
    * It grants access to it if the request access level results in no contradiction.
    * Since exclusive locks can be only held by 1 transaction, other txns have to wait until locks are released or abort and retry later.
    * As soon as the lock is released or whenever the txn terminates, the lock manager notifies one of the pending transactions, letting it acquire the lock and continue.

* *Page Cache*
    * It serves as an intermediary between persistent storage (disk) and the rest of the storage engine.
    * It stages state changes in main memory and serves as a cache for the pages that haven't been synchronized with persistent storage.
    * All changes to a database state are first applied to the cached pages.

* *Log Manager*
    * It holds a history of operations (log entries) applied to cached pages but not yet synchronized with persistent storage to guarantee that they won't be lost in case of a crash.
    * It is used to reapply those operations and reconstruct the cached state during startup.
    * These log entries can also be used to undo the applied operations in case of aborted transactions.

* Distributed transactions require additional coordination and remote execution.

**Buffer Management**
* To reduce the no. of accesses to persistent storage, *pages are cached in memory.* When page is requested again, cached copy is returned.
* The problem of caching pages is not limited to databases. OS also have the concept of page cache.

**Caching Semantics**
* All changes made to buffers are kept in memory until they are eventually written back to disk.
* As no other process is allowed to make changes to the backing file, this synchronization is a one-way process: from memory to disk and not vice-versa.
* It helps to keep the tree partially in memory without making additional changes to the algorithm and materializing objects in memory.

* If the page contents are not yet cached, the cache translates the *logical page address* or page number to its *physical page address*, loads its contents in memory, and *returns its cached version to the storage engine*.
* Once returned, the buffer with cached page contents is said to be *referenced*, and the storage engine has to hand it back to the page cache or *dereference* once it's done.

* If the page is modified, it is marked as dirty. A dirty flag set on the page indicates that its contents are out of sync with the disk and have to be flushed for durability.

**Cache Eviction**
* If the page contents are in sync with the disk (i.e. were already flushed or were never modified) and the page is not pinned or referenced, it can be evicted right away.
* *Dirty pages have to be flushed before they can be evicted.*
* Referenced page should not be flushed while some other thread is using them. 

* Since triggering a flush on every eviction might be bad for performance, some databases use a separate background process that cycles through the dirty pages that are likely to be evicted, updating their disk versions. Eg: PostgreSQL has a background flust writer.

* *Durability*: If the database that has crashed, all the data that was not flushed is lost.
* To make sure that all changes are persisted, flushes are coordinated by the *checkpoint* process.
* The checkpoint point process controls the WAL and page cache and ensures that they work in lockstep.
* WAL log records can only be discarded when the page cache is flushed.

* Trade-off b/w several objectives:
    1. Postponing flushes to reduce the no. of disk accesses.
    2. Preemptively flush pages to allow quick eviction.
    3. Pick pages for eviction and flush in the optimal order.
    4. Keep cache size within its memory bounds.
    5. Avoid losing the data as it is not persisted to the primary storage.

**Locking Pages in Cache**
* Having to perform disk I/O in each read or write is impractical: subsequent reads may request the same page, just as subsequent writes may modify the same page.
* Since B-Tree gets narrower at the top, higher-level nodes are hit for most of the nodes. Splits and merges also eventually propagate to the higher-level nodes.
* We can lock/pin pages that have a high probability of being used in nearest time.
* Since at each lower B-Tree node level, the no. of nodes grows at an exponential level, we can keep the higher-level nodes (only a small fraction of the tree) reside in memory permanently, and other parts can be paged in on demand.


* *Batching of disk writes*
    * Operations performed against a subtree may result in structural changes that contradict each other - for example, multiple delete operations causing merges followed by writes causing splits.
    * Likewise for structural changes that propagate from different subtrees (structural changes occuring close to each other in time, in different parts of the tree, propagating up).
    * These operations can be buffered together by applying changes only in memory, which can reduce the disk writes and ammortize the operation costs, since only one write can be performed instead of multiple writes.

**Prefetching and Immediate Eviction**
* The page cache also allows the storage engine to have fine-grained control over prefetching and eviction.
* It can be instructed to load pages ahead of time before they are accessed. Eg. when the leaf nodes are traversed in a page scan, the next leaves can be preloaded.

* Similarly if a maintenance process loads the page, it can be immediately evicted after the process finishes, since it's unlikely to be useful for in-flight queries. Eg. PostgrSQL uses a circular buffer (FIFO page replacement policy) for large sequential scans.

**Page Replacement**
* The choice of a page replacement algorithm has a significant impact on latency and the no. of performed I/O operations.
* *Belady's anomaly*: Increasing the no. of pages might inc. the no. of evictions if the used page algorithm is not optimal.

**FIFO and LRU**
* *FIFO*
    * Since it does not account for subsequent page accesses, only for page-in events, this proves to be impractical for most real-world systems. 
    * Eg. root and topmost-level pages are paged in first and are first candidates for eviction.
* *LRU*
    * Used again: Move to the tail of the queue.
    * *Updating references and relinking nodes can become expensive in a concurrent environment*.
* *2Q*
    * Puts pages into the first queue during the initial access and moves them to the 2nd hot queue on subsequent accesses.
    * This allows to distinguish between the frequently accessed and recently accessed pages.
* *LRU-K*
    * Identifies frequently referenced pages by keeping track of the last K accesses.

**CLOCK**
* In some situations, efficiency may be more important that precision.

**LFU**

**Recovery**
* A WAL is an append-only auxiliary disk-resident structure used for crash and transaction recovery.
* ARIES (Algorithm for Recovery and Isolation Exploiting Semantics).

**Log Semantics**
* The WAL consists of log records.
* Each log has a unique, monotonically increasing LSN (log sequence number, rep by internal counter or timestamp).
* Since log records do not necessarily occupy an entire disk block, their contents are cached in the *log buffer* and are flushed to the disk in a *force* operation.
* Forces happen as the log buffer fills up, and can be requested by the transaction manager or a page cache.

* Checkpoints are a way to know that log records upto a certain mark are fully persisted and aren't required anymore.
* *Sync checkpoint*: A process that forces all dirty pages to be flushed on disk .It fully synchronizes the primary storage structure.

* *Fuzzy checkpoints*
    * Flushing the entire contents on disk is impractical and would require pausing all running operations until the checkpoint is done, so most databases implement fuzzy checkpoints.
    * The last_checkpoint pointer stored in the log header contains the information about the last successful checkpoint.
    * A fuzzy checkpoint begins with a special begin_checkpoint log specifying its start and ends with end_checkpoint log record, containing information about the dirty pages and the contents of a transaction table.
    * Pages are flushed asynchronously and once this is done, the last_checkpoint record is updated with the LSN of the begin_checkpoint record and in case of a crash, the recovery process will start from there.

**Operation Vs Data Log**
* Some database systems use *shadow paging*: a *copy-on-write* technique ensuring data *durability* and transaction *atomicity*.
* New contents are placed into the new unpublished shadow page and made visible with a pointer flip, from the old page to the one holding new contents.
* Physical logging stores before and after images, requiring entire pages affected by this operation to be logged. A logical log specifies which operations have to be applied to the page. 
* In practice, many DBs used a combination of the above 2 approaches, using logical logging to perform an undo (for concurrency and performance) and physical logging to perform a redo (to improve recovery time).

**Steal and Force Policies**
* To determine when the changes made in memory have to be flushed on disk, DBs define steal/no-steal and force/no-force policies.
* *Steal policy*: A recovery method that allows flushing a page modified by the transaction even before the transaction has committed.
* *Force policy*: It requires all pages modified by the transactions to be flushed on disk before the transaction commits.

* Steal and force policies are important to understand since they have implications for transaction undo and redo.

* *Pros of cons of these:*
    * Using the *no-steal* policy allows implementing *recovery only using redo entries*: old copy is contained in the page on disk and modification is stored in the log.
    * With *no-force*, we potentially can buffer several updates to pages by *deferring* them. Since page contents have to be cached in memory for that time, a larger page cache may be required.
    * When force policy is used, crash recovery doesn't need any additional work to reconstruct the results of committed transactions, since pages modified by these transactions are already flushed.
        * A major drawback of this is that transactions take longer to commit due to the necessary I/O.

**ARIES**
* It is a steal/no-force recovery algorithm.
* It uses physical redo to improve performance during recovery (since changes can be installed quicker).
* It uses logical undo to improve concurrency during normal operation (since logical undo operations can be applied to pages independently).

* When the DB system restarts after the crash, recovery proceeds in 3 phases:
    1. The *analysis* phase identifies *dirty pages* and *in-progress* transactions at the *time of crash*.
        * Information about dirty pages is used to identify the starting point for the redo phase.
        * In progress transactions info is required in undo phase to roll back incomplete transactions.
    2. The *redo* phase repeats the history up to the point of a crash and restores the DB to the previous state.
        * This phase is done for incomplete transactions as well as the ones that were committed but whose contents weren't flushed to persistent storage.
    3. The *undo* phase rolls back all incomplete transactions and restores the DB to the last consistent state.
        * All operations are rolled back in reverse chronological order.
        * In case the DB crashes again during recovery, operations that undo transactions are logged as well to avoid repeating them.

* ARIES use LSNs for identifying log records, tracks pages modified by running transactions in the dirty page table, and uses physical redo, logical redo and fuzzy checkpointing.

