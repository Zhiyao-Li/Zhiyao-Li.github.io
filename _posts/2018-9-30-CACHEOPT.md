---
layout: article
title: Ten Advanced Optimizations of Cache Performance
key: 2018093001
lang: en
tags: [computer architecture a quantitative approach]
modify_date: 2018-9-30
pageview: true
---


## General guide metrics:
1. Reducing the hit time
2. Increasing cache bandwidth
3. Reducing the miss penalty
4. Reducing the miss rate 
5. Reducing the miss penalty or miss rate via parallelism

<!--more-->

### First Optimization: Small and Simple First-level Caches to Reduce Hit Time and Power
1. The critical timing path in a cache hit is the three-step process of addressing the tag memory using the index portion of the address, comparing the read tag value to the address, and setting the multiplexor to choose the correct data item if the cache is set associative.
2. In many recent processors, designers have opted for more associativity rather than larger caches: less additional consumption of hit time and energy
3. Three other reasons for choosing higher associativity:
  * Many processors take at least two clock cycles to access the cache and thus the impact of a longer hit time may not be critical.
  * To keep TLB out of the critical path, almost all L1 caches should be virtually indexed. This limits the size of the cache to the page size times the associativity.
  * With the introduction of multithreading, conflict misses can increase, making higher associativity more attractive.

### Second Optimization: Way Prediction to Reduce Hit Time
1. Way prediction means the multiplexor is set early to select the desired block, and only a single tag comparison is performed that clock cycle in parallel with reading the cache data.
2. Added to each block of a cache are block predictor bits. The bits select which of the blocks to try on the next cache access.

### Third Optimization: Pipelined Cache Access to Increase Cache Bandwidth
1. This optimization is simply to pipeline cache access so that the effective latency of a first-level cache hit can be multiple clock cycles, giving fast clock cycle time and high bandwidth but slow hits.

### Fourth Optimization: Nonblocking Caches to Increase Cache Bandwidth
1. For pipelined computers that allow out-of-order execution, the processor need not stall on a data cache miss.
2. A nonblocking cache or lookup-free cache escalates the potential benefits of such a scheme by allowing the data cache to continue to supply cache hits during a miss.
3. A subtle and complex option is that the cache may further lower the effective miss penalty if it can overlap multiple misses: a "hit under multiple miss" or "miss under miss" optimization.
4. In general, out-of-order processors are capable of hiding much of the miss penalty of an L1 data cache miss that hits in the L2 cache but are not capable of hiding a significant fraction of a lower level cache miss.
5. Deciding how many outstanding misses to support depends on a variety of factors:
  * The temporal and spatial locality in the miss stream, which determines whether a miss can initiate a new access to a lower level cache or to memory
  * The bandwidth of the responding memory or cache
  * To allow more outstanding misses at the lowest level of the cache(where the miss time is the longest) requires supporting at least that many misses at a higher level, since the miss must initiate at the highest level cache
  * The latency of the memory system

### Fifth Optimization: Multibanked Caches to Increase Cache Bandwidth
1. We can divide cache into independent banks that can We can divide cache into independent banks that can support simultaneous accesses. Banks were originally used to improve performance of main memory and are now used inside modern DRAM chips as well as with caches.
2. Sequential interleaving: banking works best when the accesses naturally spread themselves across the banks, a simple mapping that works well is to spread the addresses of the block sequentially

### Sixth Optimization: Critical Word First and Early Restart to Reduce Miss Penalty
1. This strategy is impatience: Don't wait for the full block to be loaded before sending the requested word and restarting the processor.
2. Two specific strategies:
  * Critical word first---Request the missed word first from memory and send it to the processor as soon as it arrives; let the processor continue execution while filling the rest of the words in the block.
  * Early restart---Fetch the words in normal order, but as soon as the requested word of the block arrives send it to the processor and let the processor continue execution.
  * Generally, these techniques only benefit designs with large cache blocks, since the benefit is low unless blocks are large. Note that caches normally continue to satisfy accesses to other blocks while the rest of the block is being filled.
  * The benefits of critical word first and early restart depend on the size of the block and the likelihood of another access to the portion of the block that has not yet been fetched.

### Seventh Optimization: Merging Write Buffer to Reduce Miss Penalty
1. Write-through caches rely on write buffers, as all stores must be sent to the next lower label of the hierarchy. Even write-back caches use a simple buffer when a block is replaced.
2. If the write buffer is empty, the data and the full address are writing in the buffer, and the write is finished from the processor's perspective; the processor continues working while the write buffer prepares to write the word to memory.
3. Write merging: If the buffer contains other modified blocks, the addresses can be checked to see if the address of the new data matches the address of a valid write buffer entry. If so, the data are combined with that entry.
4. If the buffer is full and there is no address match, the cache(and processor) must wait until the buffer has an empty entry. This optimization uses the memory more efficiently since multiword writes are usually faster than writes performed one word at a time.
5. The optimization also reduces stalls due to the write buffer being full. e.g. Assume we had four entries in the write buffer, and each entry could hold four 64-bit words. Without this optimization, four stores to sequential addresses would fill the buffer at one word per entry, even though these four words when merged exactly fit within a single entry of the write buffer.

### Eighth Optimization: Complier Optimizations to Reduce Miss Rate
1. Loop Interchange: Some programs have nested loops that access data in memory in nonsequential order.
2. Blocking: This optimization improves temporal locality to reduce misses.
  * The number of capacity misses clearly depends on N and the size of the cache. If it can hold all three N-by-N matrices, then all is well, provided there are no cache conflicts. If the cache can hold one N-by-N matrix and one row of N, then at least the ith row of y and the array z may stay in the cache. Less than that and misses may occur for both x and z. In th worst case, there would be O(N3) operations.
  * To ensure that the elements being accessed can fit in the cache, the original code is changed to compute on a submatrix of size B by B. Two inner loops now compute in steps of size B rather than the full length of x and z. B is called the blocking factor. (Assume x is initialized to zero)

### Ninth Optimization: Hardware Prefetching of Instructions and Data to Reduce Miss Penalty or Miss Rate
1. Nonblocking caches effectively reduce the miss penalty by overlapping execution with memory access. Another approach is to prefetch items before the processor requests them.
2. Instruction prefetching is frequently done in hardware outside of the cache.
3. Prefetching relies on utilizing memory bandwidth that otherwise would be unused, but if it interferes with demand misses it can actually lower performance.

### Tenth Optimization: Compiler-Controlled Prefetching to Reduce Miss Penalty or Miss Rate
1. An alternative to hardware prefetching is for the compiler to insert prefetch instructions to request data before the processor needs it.
2. Two flavors of prefetch:
  * Register prefetch will load the value into a register
  * Cache prefetch loads data only into the cache and not the register
3. The most effective prefetch is "semantically invisible" to a program: It doesn't change the contents of registers and memory, and it cannot cause virtual memory faults. Most processors today offer nonfaulting cache prefetches.
4. Prefetching makes sense only if the processor can proceed while prefetching the data; that is, the caches do not stall but continue to supply instructions and data while waiting for the prefetched data to return. The data cache for such computers is normally nonblocking.
5. Issuing prefetching instructions incurs an instruction overhead, however, compilers must take care to ensure that such overheads do not exceed the benefits. By concentrating on references that are likely to be cache misses, programs can avoid unnecessary prefetches while improving average memory access time significantly.