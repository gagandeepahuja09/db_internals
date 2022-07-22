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