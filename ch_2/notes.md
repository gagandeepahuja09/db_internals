**B-Tree Basics**

* Why should we consider alternative to traditional trees? eg. BST, 2-3 trees and AVL trees.

**Tree Balancing**
* A balanced tree is defined as one that has a height of log2 N and the difference in the height of the 2 subtrees <= 1.

* Instead of adding new elements to one of the tree branches and making it longer, *the tree is balanced after each operation*.
* Balancing is done by reorganizing nodes in a way that minimizes tree height and keep the number of nodes on each side within bounds.

* One of the ways to keep the tree balanced is to perform rotation steps are nodes are added or removed.
* If the insert operation leaves a branch unbalanced (two consecutive nodes in the branch have only one child), we can rotate the nodes around the middle one.

**Tress for Disk-Based Storage**
* Fanout: Maximum allowed no. of children per node.
* Fanout and height are inversely related.

* **Problems with BST**
    * In case of BSTs due to *low fanout*, we have to *perform balancing, relocate nodes, and update pointers* rather frequently. Increase maintainence cost makes them impractical as on-disk data structures.
    * *Locality*: Since elements are added in a random order, there's no guarantee that a newly created node is written close to its parent. This means that *node-child pointers may span across several pages*. We can improve this to certain index by modifying the tree layout and using paged binary trees.
    * Due to *larger heights* of log2 N, we have to perform these many *seeks* to locate the searched element and subsequently perform the same no. of *disk transfers*. 

* Better suited for disk implementation:
    1. High fanout: To improve locality of the neighboring keys.
    2. Low height: To reduce the no. of seeks during traversal.

**Disk-Based Structures**

**Hard Disk Drives**
* On spinning disks, seeks increase cost of random reads because they require *disk rotation* and *mechanical head movements* to *position the read/write head to the desired location*.
* Once the expensive part is done, reading or writing continguous (sequential operations) bytes is relatively cheap.

* The smallest transfer unit of a spinning drive is a *sector*. So, when some operation is performed, an entire sector can be read / written. Sector size => 512 bytes to 4 kB.

**Solid State Drives (SSDs)**
* They *don't have moving parts*. There is no disk that spins or head that has to be positioned for the read.

* A typical SSD is built of memory cells, connected into strings (32-64 cells per string).
* Strings are combined into arrays, arrays are combined into pages and pages are combined into blocks.
* *Smallest unit* that can be written or read is a *page*.