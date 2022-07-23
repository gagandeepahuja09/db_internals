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