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
    * Multiple concurrently executing transactions should be able to run without interference as it there were no other transactions executing at the same time.
    * Isolation defines *when* the changes to the DB state may become visible, and *what* changes may become visible to the concurrent transactions.
    * Many DBs use default isolation levels that are weaker than the given definition for performance reasons.
    * Depending on the methods and approaches used for concurrency control, changes made by a transaction may or may not be visible to other transactions.

* *Durability*:
    * Once a transaction has been committed, all DB state modifications have to be persisted on disk and be able to survive power outages, system failures and crashes.

* Implement transactions in a DBMS, in addition to a storage that organizes and persists data on disk requires several components to work together.

* *Transaction Manager*
    * It *coordinates, schedules and tracks* transactions and their individual steps.

* *Lock Manager*
    * Whenever a lock is requested, the lock manager checks if it is already held by any other transaction in shared or exclusive mode.
    * It grants access to it if the request access level results in no contradiction.
    * Since exclusive locks can be only held by 1 transaction, other txns have to wait until locks are released or abort and retry later.
    * As soon as the lock is released or whenever the txn terminates, the lock manager notifies one of the pending transactions, letting it acquire the lock and continue.