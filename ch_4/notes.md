**Implementing B-Trees**

1. Organization:
    * How to establish relationship between keys and pointers.
    * How to implement headers and links between pages.

2. Processes that occur during root-to-leaf descends:
    * Performing binary search.
    * Collecting breadcrumbs.
    * Keeping track of parent nodes in case we later need to split or merge nodes.

3. 
    * Optimization Techniques:
        * Rebalancing
        * Right-only appends
        * Bulk-loading
    * Maintainence processes
    * Garbage collection

**Page Header**
* They hold information about the page that can be used for *navigation, maintainence, and optimizations*.
* It usually contains flags that describe:
    * Page contents and layout.
    * Number of cells in the page.
    * Lower and upper offsets marking the empty data. (used to append cell offsets and data).
* PostgreSQL stores the page size and layout version in the header.
* MySQL InnoDB: no. of heap records, level, etc.
* SQLite: no. of cells, a rightmost pointer.

**Magic Numbers**
* It's usually a multibyte block, containing a constant value that can be used to signal that it is a page, specify its kind and version.
* Eg. 50 41 47 45 (hex for PAGE)
* They are often used for validation and sanity checks.

**Sibling Links**
* These links help to locate neighbors without having to ascend back to parent (which might be required all the way up to root). Could be useful for range queries.
* This approach adds some complexity to the split and merge operation as the sibling offsets have to be updated as well. Eg. when a non-rightmost sibling is split, its right sibling's backward pointer has to be re-bound to point to the newly created node.
* Since updates have to happen in a sibling node, not in a splitting/merging node, it may require additional locking.

**Overflow Pages**
* Node size and tree fanout values are fixed and do not change dynamically. It would also be difficult to come up with a value that would be universally optimal.
* The B-Tree algorithm specifies that every node keeps a specified no. of items.
* Since some values have different sizes, we may end with a situation where according to the B-Tree algorithm, the node is not full yet, but there's no more free space on the fixed-size page that holds this node.

* Resizing the page requires copying already written data to the new region and is often impractical. We can instead build nodes from multiple linked pages. 
* Default page size is 4kB. Instead of allowing arbitrary size, nodes are allowed to grow in 4kB increments.

* Most B-Tree implementations allow storing only up to a fixed no. of payload bytes in the B-Tree node directly and spilling the rest to the overflow page.
* This value is calculated by dividing the node size by fanout.
* Using this approach, we cannot end up in a situation where the page has no free space as it will always have at least max_payload_size bytes.
* Overflow page's page ID is stored in the header of the parent page.

* Since keys usually have high cardinality, storing a portion of a key makes sense, as most of the comparisons can be made on the trimmed part that resides in the primary page.

**Binary Search**
* Positive number signifies the position of the number. Negative number specifies that the number was not found. Its absolute value tells us the position at which it should be inserted.

**Propagating Splits and Merges**
* B-Tree splits and merges can propagate to higher levels. For this, B-Tree nodes may have parent pointers.
* Just like sibling-pointers, parent pointers have to be updated when the parent changes.
* Some implementations use parent-pointers for leaf traversals to avoid sibling pointers which may result in deadlock.

**Breadcrumbs**
* Instead of storing and maintaining parent node pointers, we can keep track of nodes traversed on the path, and follow the chain of parent nodes in reverse order. (cascading splits or merges).
* The most natural data structure for breadcrumbs is a stack. Eg. PostgreSQL BTStack. This stack is maintained in memory.

**Rebalancing**
* Some B-Tree implementations attempt to postpone their split and merge operations to ammortize their costs.
* This is done by rebalancing elements within the level or by moving elements from more occupied nodes to less occupied ones for as long as possible before finally performing a split or merge.

* This helps to improve the *node occupancy* and may reduce the *no. of levels* within a tree at a potentially higher *maintainence cost of rebalancing*.

* Since balancing changes the min/max invariant of the sibling nodes, we have to update keys and pointers to the parent node to preserve it.