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