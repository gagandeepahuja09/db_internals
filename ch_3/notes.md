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