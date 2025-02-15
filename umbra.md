# Umbra: A Disk-Based DBMS with In-Memory Performance

URLs: [Paper](https://db.in.tum.de/~freitag/papers/p29-neumann-cidr20.pdf) (2020), [Presentation](https://www.youtube.com/watch?v=pS2_AJNIxzU) (2022)

## 1. Introduction

**Umbra** is a next-generation database system from Thomas Neumann’s group at TU Munich. It represents an evolution of **HyPer** (their previous in-memory database) toward a disk-based design that still attains near in-memory performance. While pure in-memory DBMSs deliver high speed when data fits in RAM, memory remains expensive and has not grown as quickly as once predicted. Meanwhile, **SSDs** have become fast (read bandwidth in GB/s) and relatively cheap. Umbra’s goal is to exploit **large RAM buffers** plus **fast SSDs** to achieve:

1. **High performance** when the working set is in memory.
2. **Transparent, graceful scaling** beyond memory when needed.

Umbra’s central innovation is a **buffer manager** that eliminates much of the overhead typical of disk-based systems. It supports **variable-size pages** to simplify data structure management, combined with concurrency control strategies that minimize locking overhead. The net effect is an engine that feels like an in-memory system when memory is abundant but continues to scale when data exceeds RAM capacity.

---

## 2. Why Umbra?

1. **Growth of DRAM Has Slowed**  
   A few years ago, single servers with “large” memory (e.g., multiple TB) seemed feasible. However, the cost and availability of multi-terabyte DRAM have not improved fast enough. Maintaining entirely in-memory data can be cost-prohibitive.

2. **SSDs Offer High Throughput & Lower $/GB**  
   Modern NVMe SSDs can reach several GB/s bandwidth at a fraction of DRAM’s cost. Combining a **large, but not extreme** RAM buffer with SSD-based backing storage provides a robust scaling approach.

3. **Pure In-Memory Systems Are Non-Elastic**  
   As soon as data no longer fits, performance in a purely in-memory system can collapse. A disk-based design gracefully accommodates bigger data sets but typically imposes overhead. Umbra is carefully designed to **retain in-memory speeds** for working-set accesses while seamlessly handling out-of-memory cases.

---

## 3. Core Architectural Ideas

Umbra inherits many ideas from **HyPer** (e.g., compiled queries, parallel pipelined execution) but introduces new concepts to be truly disk-based with minimal overhead. Major changes include:

1. **Buffer Manager with Variable-Size Pages**  
2. **Versioned Latches & Pointer Swizzling**  
3. **Adaptive Compilation & State-Machine Execution**  
4. **Online Statistics Gathering**  
5. **Flexible String Handling**  

We highlight each below.

### 3.1 Buffer Manager & Variable-Size Pages

Traditional disk-based systems use **fixed-size pages** (e.g., 4 KB or 16 KB). This simplifies the buffer manager but complicates everything else:

- Large data items (e.g., long strings, dictionaries) must be split across multiple pages or “chained,” complicating data structure logic.
- Reconstructing larger items across pages incurs overhead.

**Umbra** flips this approach. It uses **variable-size pages** and can store, for instance, a 64 KB or even multi-MB data object in one page. Data structures like dictionaries or large strings can remain contiguous in memory, simplifying usage and speeding lookups.

#### Virtual vs. Physical Memory
Umbra’s buffer manager exploits modern OS features:
- It reserves large contiguous **virtual address** blocks for each “size class” (e.g., 64 KB, 128 KB, …).
- A page is mapped to physical RAM only when actively loaded.  
- When evicting, Umbra can unmap physical pages without losing the stable virtual address.

This strategy **prevents external fragmentation**: data can live in a single “page” of the right size, and the OS’s virtual memory mapping handles potential physical fragmentation behind the scenes.

### 3.2 Concurrency Control in the Buffer Manager

Even if data is in memory, a naive buffer manager can add serious overhead (locks, hashing, etc.). Umbra addresses concurrency overhead via:

1. **Pointer Swizzling**  
   - Instead of a global hash table to map page IDs → addresses, each “swip” (64-bit pointer) either holds a direct pointer or an on-disk page ID.  
   - If a page is in memory, the swip is a pointer (fast direct access). If it’s on disk, the swip indicates how to load it.  
   - Upon eviction, Umbra changes the swip back to a disk ID as needed.

2. **Versioned Latches**  
   - Each buffer-managed page has a **64-bit version counter + state bits**.  
   - Writers acquire an exclusive latch, bump the version on commit. Readers can latch **optimistically**, reading version counters to detect conflicts or changes.  
   - This drastically reduces lock overhead in read-heavy or scan-heavy workloads while allowing safe updates.

### 3.3 B-Tree Storage

Umbra organizes each table’s storage in a **B+-tree** keyed on a synthetic, strictly increasing tuple ID:

- **Leaf pages** typically use ~64 KB pages. Large tuples (or large strings) can occupy bigger pages.  
- B-tree nodes do **not** chain leaf siblings to ensure that each page has exactly one “owning swip.” This property simplifies pointer swizzling and eviction logic.  
- Scanning is handled by holding an optimistic latch on the parent and quickly navigating to next leaves.

### 3.4 Hybrid Multi-Versioning

Umbra provides **transaction isolation** with multi-version concurrency. However, writing all version data to disk is expensive. The system uses:

- **In-Memory Version Buffers** for small transactions: a pointer to the in-memory “delta” suffices.  
- **On-Page Markers** for large bulk operations (e.g., large update or delete). The page itself stores bits for created/deleted states, so no separate giant memory structure is required.

---

## 4. Further Changes for an SSD-Based System

### 4.1 String Handling

In a disk-based environment, references inside pages can become invalid upon eviction. Umbra solves this with **three storage classes** for strings:

- **Persistent**: e.g., query constants. Remain valid for entire DB uptime.  
- **Transient**: data from a B-tree leaf is pinned short-term; if the page is evicted, references become invalid. Umbra copies strings if needed for a longer scope.  
- **Temporary**: ephemeral strings created in query pipelines (e.g. function results). Freed after use.

The system thus avoids breakage if pages get evicted, at minimal overhead.

### 4.2 Statistics & Sampling

In-memory systems might re-scan a table to gather updated stats. For disk-based operation, scanning is expensive. Instead, Umbra:

- Maintains **online reservoir sampling** for each table. As inserts/deletes happen, it updates the sample in memory with minimal overhead.  
- Tracks **HyperLogLog** sketches per column to estimate distinct values.  
- This combination yields robust cardinality estimates cheaply, crucial for cost-based optimization.

### 4.3 Adaptive Compilation & State-Machine Execution

Umbra compiles queries into specialized code, similarly to HyPer. But it refines the approach:

1. **State Machine Pipelines**  
   - Each query pipeline is broken into smaller “steps,” each compiled as a function.  
   - A “step” might be single- or multi-threaded. The system orchestrates steps in a dataflow manner.  
   - This design lets Umbra **suspend** or **deschedule** queries easily (helpful if we need resources for short queries or are waiting on I/O).

2. **Adaptive Compilation**  
   - Umbra first compiles each step into a quick, “low-latency” format.  
   - If runtime monitoring shows the step is very hot (it runs many times or processes lots of data), Umbra triggers a more expensive LLVM-based optimization in the background.  
   - Once done, it switches to the optimized code. Short steps never pay the cost of heavy compilation; big steps get high-speed code.

---

## 5. Performance Highlights

### 5.1 Competitive with In-Memory DBs

On the **Join Order Benchmark (JOB)** and **TPC-H** (scale factor 10) with warm caches, Umbra matches or surpasses HyPer’s speed, often 1.5–3× faster. The improvement partly comes from better statistics or better compilation strategies. Crucially, Umbra’s overhead from the buffer manager is negligible when data fits.

### 5.2 Graceful I/O Scaling

When memory is insufficient, Umbra can saturate **NVMe SSD** bandwidth. Microbenchmarks scanning large tables show:

- Umbra quickly hits 1+ GB/s read rates.  
- The overhead of B-trees, pointer swizzling, version latching, etc. remains minimal.

Hence, Umbra can shift from in-memory speed to near-SSD bandwidth-limited speed automatically, with no catastrophic slowdown.

---

## 6. Conclusion

**Umbra** is a disk-based DBMS that achieves near in-memory performance for the “hot” working set while seamlessly handling out-of-memory cases on **SSDs**. Key design elements include:

- **Variable-Size Pages**: Freed the rest of the system from the old fixed-page constraints and chaining logic.  
- **Low-Overhead Concurrency**: Pointer swizzling plus versioned latches eliminate global hash tables and reduce lock contention.  
- **Adaptive Query Compilation**: Minimizes compilation overhead while generating fast code for expensive parts of the query.  
- **Online Sampling & HLL Sketches**: Provides accurate cardinality estimates without incurring large random I/O.  

Umbra aspires to combine the best aspects of pure in-memory designs (high performance, simpler data structures) with the robust scaling and cost effectiveness of a disk-based system, bridging the memory gap with a well-designed buffer manager that is effectively invisible when data is in memory.