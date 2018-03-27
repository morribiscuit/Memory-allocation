## 1 Introduction

General-purpose memory allocators can become bottlenecks in high-performance applications, so custom memory managers can be written to fit a particular application's memory usage.
For this assignment, you will implement a dynamic memory allocator.
This will provide insight into how heap memory management for an application is performed and reasoning behind allocator desgin.

## 2 Background

Every process has its own virtual address space. Parts of this space are mapped to physical memory locations through address translation, which is managed by the memory management unit and the kernel. 
You have seen first-hand in homework 0 that the address space is divided into several parts, primarily the stack, heap, and segments for code and static data.

### sbrk

Initially, the application's heap will not be allocated. The sbrk() function is what you will be using to increase the mapped region of the heap.
The syscall has the following prototype:  
```void *sbrk(int increment)```

The function manipulates the position of the break, specifically by incrementing its position by a specified number of **bytes**. 
It returns the address of the **previous** break; so you can use ```sbrk(0)``` to get the current position of the break.
Read the man page for more information.

## 3 Buddy Allocator

The buddy memory allocation algorithm works by dividing memory partitions of specific sizes in order to fit the requested memory size suitably. The Linux kernel uses a modified version of the buddy system.  

Consider a binary buddy system where each block is a power of two. You start off with a single block of some maximum block size, say 2<sup>n</sup>. 
The system works by subdividing this block to best fit the sizes of memory allocations to powers of 2.
A block's buddy is its immediate neighbour formed during a split. Buddy blocks are merged when both are free, this reduces external fragmentation.

Whenever a request for a memory allocation is performed:
- The size of the request is rounded up to the next power of two, say 2<sup>k</sup>
- Search for the most suitable block size of 2<sup>k</sup>
- If found, allocate the memory block
- If not found:
    - Split a free memory block larger than the requested memory size in half
    - Repeat until at the correct block size of 2<sup>k</sup>
    
Whenever memory is freed:
- Ensure the memory location is valid to be freed
- If the block's buddy is free:
    - Merge the two blocks
    - Attempt to merge the buddy of the merged block until the upper block limit is reached or the buddy is not free

The main advantage of buddy allocation is fast block merging, since 'buddy' blocks are coalesced without needing to traverse the entire list looking for free blocks.
The main disadvantage of buddy allocators is that they suffer from internal fragmentation, because allocations are rounded up to the nearest block size. 
For example, a 68-byte allocation will require a 128-byte block.

### 4.2 Allocation

Implement buddy memory allocation as described above. We have defined a block structure for you to use in mm_alloc.h

```
void* mm_malloc(size_t size)
```
The function takes in the reqeusted memory size in number of bytes (where size_t represents an unsigned integer type) and returns a pointer to a valid memory location with at least that many bytes available.  

Return NULL if the requested size is zero or cannot be allocated.

- **Implement the mm_malloc() function using the buddy allocation algorithm**

NOTES:
- We have predefined the maximum block size, MAX_SIZE, to be 1 MiB. 
- Remember to add the sizeof your block struct to the requested size; you need to fit the block metadata into the same space.  
- On the first mm_malloc request, extend the heap with ```sbrk()``` and create the maximum block. Afterwards, you can follow the normal sequence of allocating a block.  
- Use a first-fit algorithm when searching for a block: Search for the first block greater or equal in size to allocate, and split the block as usual if greater.  
- When initialising a block, initialise the 'merged_buddy' array elements to NULL.
- When splitting a block, add the buddy of the block being split to the 'merged_buddy' arrays of the split blocks. This aids in the merging process.
- Beware of pointer arithmetic behaviour!

### 4.3 Deallocation

```
void mm_free( void* ptr )
```  
The function takes in a pointer to a memory block previously allocated using mm_alloc() and frees it, making it available for allocation again.
Ensure that only pointers to valid blocks are freed. If ptr is NULL, do nothing.
To reduce fragmentation and use memory efficiently, blocks should be merged when when freed.  

- **Implement the mm_free() function**
- **Implement block merging on free**

NOTES:
- When a block is freed, check its buddy. If the buddy is also free, merge the blocks together.  
- If the buddy is not the same size as the current block, obviously it cannot be merged since it is split.
- Check on the larger merged blocks until the buddy is not free, or at the largest block size.  
- You should use the 'merged_buddy' array to hold the buddies of the merged blocks.
- Keep the 'merged_buddy' array updated as blocks are split and merged.

### 4.4 Reallocation

```
void* mm_realloc(void* ptr, size_t size)
```
The function takes in a pointer to a memory block previously allocated using mm_alloc() and returns a pointer to the data in a block of at least size size.  

- **Implement the mm_realloc() function**
