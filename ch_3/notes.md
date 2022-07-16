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

**Binary Encoding**
* To store data on disk, it needs to be encoded using a format that is compact and easy to serialize and deserialize.
* We discuss the main principles for creating efficient page layouts. These principles apply to any binary format: we can use similar guidelines to create file and serialization formats or communication protocols.

* Before we can organize records into pages, we need to understand:
    1. How to represent keys and data records in binary format.
    2. How to combine multiple values into more complex structures.
    3. How to implement variable-size types and arrays.

**Primitive Types**
* Use the same byte-order (endianness) for both encoding and decoding.
    * Big-endian: MSB to LSB.
    * Little-endian: LSB to MSB.
* Floating point numbers are represented by their sign, fraction and exponent.
* A 32-bit float represents a single-precision value.
    * The first 23 bits represent a fraction.
    * The following 8 bits represent an exponent.
    * 1 bit represents a sign.
    * Since a floating-point value is calculated using fractions, the no. this representation yields is just an approximation.

**Strings and Variable Size Data**
* All primitive numeric types have a fixed size.
* Composing more complex values together is much like struct in C.
`
    String
    {
        size uint_16
        data byte[size]
    }
`
* Strings and other variable-size data types can be serialized as a number, representing the length of the array string, followed by size bytes: the actual data.
* The above representation is called Pascal string. An alternative to it is null-terminated strings.
* Pascal string has several advantages:
    1. Find length of string in constant time.
    2. A language specific string can be composed by slize `size` bytes from memory and passing the byte array to a string constructor.

**Bit-Packed Data: Booleans, Enums and Flags**
* Using an entire byte a boolean's representation could be useful. Developers often batch boolean values in groups of 8 to solve this problem.

**General Principles**
* Most in-place update storage structures use pages of the same size, since it significantly simplifies read and write access.
* Append-only storage structures often write data page-wise too: records are appended one after the other and as soon as the page fills up in memory, it is flushed on disk.

* The file usually starts with a fixed-size header and may end with a fixed-size trailer.
* They hold auxiliary information that should be accessed quickly or is required for decoding the rest of the file.
* The rest of the file is split into pages.

* Many data stores have a fixed schema, specifying the number, order and type of fields the table can hold. Having a fixed schema allows to reduce the disk space. Instead of repeatedly writing field names, we can use their positional identifier.

* We could store the fixed-size fields in the head of the structure followed by the variable-size ones.

`
    Fixed-size fields:
    (4 bytes) employee_id
    (4 bytes) tax_number
    (2 bytes) first_name_length
    (2 bytes) last_name_length

    Variable-size fields:
    (first_name_length bytes) first_name
    (last_name_length bytes) last_name
`

* To access first_name, we can slice first_name_length after the fixed sized area.
* We can locate last_name by checking the size of the variable size fields that precede it.
* To avoid calculations involving multiple fields, we can encode both offset and length to the fixed-size areas.

**Page Structure**
* Size of multiple file system blocks. 4-16 kB.
* From a structure perspective, we distinguish b/w the leaf nodes that hold keys and data record pairs, and non leaf nodes that hold keys and pointers to other nodes.

* Each B-Tree node occupies one page or multiple pages linked together. So, in context of B-trees the term node and page are used interchangably.

* Page organization:
    p0 | k1 | v1 | p1 | k2 | v2 | .... | pn | kn | vn | ...unused

* Downsides of above approach:
    1. Appending a key anywhere but the right side requires relocating elements.
    2. It doesn't allow managing or accessing variable-size records efficiently and works only for fixed-size data.

**Slotted Pages**
* When storing *variable-size records*, the main problem is *free-space management*: reclaiming the space occupied by removed records.
* If we attempt a put a record of size n into the space previously occupied by a record of size m, unless m == n or we find another record of size m - n, the space will remain unused.

* To simplify space management for variable-size records, we can split the page into fixed-size segments. However, we end up wasting space with that too. Eg: segment size of 64 bytes, unless record size is a multiple of 64, we wast 64 - (n%64) bytes.

* To efficiently store variable-size records such as strings, binary large objects (BLOBs), etc we can use slotted pages as an organization technique. This approach is used by many databases. Eg: PostgreSQL.

* We organize the page into a collection of cells and split out the pointers and cells into 2 independent memory regions residing on different sides of the page.
* This means that we only need to reorganize the pointers pointers addressing the cells and deleting a cell can be done by nullifying the pointer or removing it.

* Advantages of slotted pages:
    1. *Minimal overhead*: The only overhead is the pointer array.
    2. *Space reclamation*: Space can be reclaimed by defragmenting and rewriting the page.
    3. *Dynamic Layout*: From outside the page, slots are referenced only by their IDs, so the exact location is internal to the page.

**Cell Layout**
* Key cells and Key-value cells.
* All cells within a page are uniform. Eg: all cells can hold either just keys or both keys and values. Similarly all cells can hold either fixed-size of variable-size data but not a mix of both.
* This means we can store metadata describing cells once on the page level, instead of duplicating in every cell.

* To compose a key cell, we need to know:
    * Cell type (can be inferred from the page metadata)
    * Key size
    * ID of the child page this cell is pointing to
    * Key bytes

* A variable-size key cell layout looks like this (a fixed-one would have no specifier on the cell level):

`
    0               4               8
    [int]key_size   [int]page_id    [bytes]key
`

* We can group fixed-size data fields together at the start, since all fixed-size fields can be accessed by using static, precomputed offsets, and we need to calculate offsets only for the variable-size data.

* Key-value cells:
    * Cell type (can be inferred from page metadata)
    * Key size
    * Value size
    * Key bytes
    * Data record bytes

* Distinction between *offset and page ID*. 
    * Since pages have a fixed size and are managed by the *page cache*, we only need to store the page ID, which is later translated to the actual offset in the file using the lookup table.
    * Cell offsets are page-local and relative to the page start offset.

**Combining Cells into Slotted Pages**
* Keys can be inserted out of order and their logical sorted order is kept by sorting cell offset pointers in key order.
* This allows appending cells to the page with minimal effort, since cells don't have to be relocated during insert, update or delete operations.