* We access the disk in a way that is different from how we access main memory: from an application developer's perspective, memory accesses are mostly transparent.
* Because of virtual memory, we do not have to manage offsets manually.
* Disks are accessed using system calls. 

* We usually have to specify the offset inside the traget file, and then interpret on-disk representation into a form suitable for main-memory.
* We have to come up with a file format that's *easy to construct, modify and interpret*.
* We'll discuss general principles and practices that help us to design all sorts of non-disk structures, not only B-trees.

* We should think of on-disk B-trees as a *page management mechanism*. Algorithms have to *compose and navigate pages*. Pages and pointers to them have to be calculated and placed accordingly.
* Since most of the complexity in B-trees comes from mutability, we discuss details of page layouts, splitting, relocations, etc.
* Most of the complexities in LSM trees comes from sorting and maintainence. 

**Motivation**
* Creating a file format is in many ways similar to how we create data structures in languages with an unmanaged memory model.
* We allocate a block of data and slice it in any way we like, using fixed-size primitives and structures.
* If we want to reference a larger chunk of memory or a structure with variable size, we use pointers.

* Languages with an unmanaged memory model allow us to allocate more any time we need (within reasonable bounds) without us having to worry about:
    1. Whether or not there's a *contiguous memory segment available*.
    2. Whether or not it is *fragmented*.
    3. What happens after we free it.
* On disk, we have to take care of fragmentation and garbage collection ourselves.

* *Data layout* is much less important in memory than on disk.
* For a disk-resident structure to be efficient, we need to lay out data is disk that allows quick access to it.
* We'll need to come up with binary data formats and find a means to serialize and deserialize data efficiently.