# I. Performance Engineering & High Throughput Systems (Principal Engineer Depth)

This chapter covers:

- CPU architecture & pipelining basics  
- JVM hotspots & JIT internals  
- Garbage Collector internals & tuning  
- Memory leak detection  
- Threading vs async performance  
- Lock contention, atomics, CAS  
- Queues, backpressure, batching  
- Kernel & networking performance tuning  
- Flamegraphs, profiling tools  
- High throughput patterns (Uber, Meta, Netflix style)  

---

# 1. CPU ARCHITECTURE (FOR BACKEND ENGINEERS)

### Important CPU concepts:

- L1/L2/L3 caches  
- cache lines (64 bytes)  
- branch prediction  
- pipelining  
- instruction reordering  
- NUMA domains  
- CPU affinity  

---

## 1.1 Cache Hierarchy Diagram

```
CPU Core
  ├── L1 Cache (32KB)
  ├── L2 Cache (256KB)
  └── Shared L3 Cache (8–64MB)
```

**Cache misses destroy performance.**

### False sharing

Two threads write to variables on same cache line → thrash.

Fix:
- `@Contended` (Java annotation)
- add padding  

---

# 2. JVM PERFORMANCE FOUNDATIONS

JVM is divided into:

```
Heap:
  Young Gen (Eden, Survivor)
  Old Gen

Non-heap:
  Code cache
  Metaspace
  Thread stacks
```

---

## 2.1 JIT (Just-In-Time Compiler)

Hotspot JIT compiles frequently used bytecode to **native machine code**.

Phases:

1. Interpret  
2. Identify hot methods  
3. Compile with C1  
4. Optimize with C2 JIT  

---

## 2.2 Escape Analysis

JIT decides:

- object can stay on stack  
- eliminate allocations  
- remove locks  

Result:

```
new Object() → stack allocation (no GC hit)
```

---

# 3. GARBAGE COLLECTION (EXTREMELY DEEP)

GC types:

- Serial GC  
- Parallel GC  
- CMS (deprecated)  
- **G1 (default Java 11+)**  
- **ZGC (low latency)**  
- **Shenandoah (low latency)**  

---

## 3.1 G1 GC Internals

Heap divided into **regions** (1–32 MB each).

GC steps:
- concurrent marking  
- evacuation  
- refinements  

Latency target configurable:

```
-XX:MaxGCPauseMillis=200
```

---

## 3.2 ZGC Internals (Sub-10ms pauses)

ZGC uses:

- colored pointers  
- concurrent relocation  
- load barriers  

Best for:
- trading systems  
- high throughput APIs  
- real-time stream processors  

GC pause ≈ **<1ms** even at 100GB heaps.

---

# 3.3 When to Tune GC

Symptoms:
- GC >20% CPU  
- large pause spikes  
- long tail latency (p99/p999)  
- full GC events  

Tools:
- GC logs  
- JDK Flight Recorder  
- GCeasy.io  

---

# 4. MEMORY LEAK DETECTION

Even GC languages leak memory (via:

- long-lived references  
- static collections  
- caches that don’t expire  
- thread locals  
- coroutines stuck with context  

Tools:
- MAT (Memory Analyzer Tool)  
- JFR (Flight Recorder)  
- YourKit  
- Heap dumps  

---

# 5. THREADING VS ASYNC PERFORMANCE

### Myth:
“More threads = more throughput.”  
No. More threads = context switching overhead.

Thread context switch ≈ **1–2 microseconds**.  
Coroutine context switch ≈ **100–200 nanoseconds**.

---

# 5.1 Blocking Model (Thread per request)

Good when:
- CPU heavy  
- Isolation needed  

Bad when:
- high I/O wait  
- large number of concurrent clients  

---

## 5.2 Async Model (Event loop)

Used by:
- Netty  
- Node.js  
- Kotlin coroutines  

Pros:
- huge concurrency  
- minimal overhead  

Cons:
- complex code  
- backpressure needed  

---

# 6. LOCK CONTENTION & ATOMIC OPERATIONS (DEEP)

### Locks cause:

- kernel scheduling
- thread blocking
- context switching

Use Lock-Free structures when possible:
- ConcurrentLinkedQueue  
- LongAdder (sharded atomic)  
- AtomicReference  

CAS (Compare-And-Set) is faster than locks unless high contention.

---

# 7. BATCHING & QUEUEING (HIGH-THROUGHPUT SECRET)

Batching = #1 technique for higher throughput.

Examples:
- batch DB writes  
- batch Kafka sends  
- batch network writes  

Uber achieved **10x throughput** by batching Redis queries in pipelines.

---

## 7.1 Queue Backpressure Signals

Indicators:
- queue length > threshold  
- producer throughput > consumer throughput  
- thread pool saturation  
- database connection pool saturation  

Solutions:
- apply rate limiting  
- drop low priority requests  
- implement bounded queues  

---

# 8. KERNEL TUNING (LINUX PERFORMANCE)

### 8.1 File descriptors

```
ulimit -n 1000000
```

### 8.2 TCP Backlog

```
net.core.somaxconn = 65535
```

### 8.3 TCP Reuse/Recycling

```
net.ipv4.tcp_tw_reuse = 1
```

### 8.4 Socket Read/Write Buffers

Tune for high throughput systems.

---

# 9. NETWORK PERFORMANCE

Understand:

- RTT  
- bandwidth-delay product  
- Nagle’s algorithm  
- TCP slow start  
- packet loss → throughput collapse  

### Disable Nagle when needed:

```
TCP_NODELAY = true
```

---

# 10. PROFILING & FLAMEGRAPHS

Tools:
- perf  
- async-profiler  
- JFR  
- YourKit  

Flamegraph example:

```
main
 ├── handleRequest
 │     ├── parseJson
 │     ├── service
 │     └── databaseCall
 └── GC
```

Hot bars indicate bottleneck.

---

# 11. COMMON JAVA/KOTLIN PERFORMANCE BUGS

- creating objects inside loops  
- using Streams in hot path  
- excessive logging  
- using synchronized unnecessarily  
- using blocking calls inside coroutines  
- excessive context switching  
- unbounded thread pools  
- HTTP client without pooling  
- ORM fetching too much data  

---

# 12. DATABASE HOT SPOTS

## 12.1 Hot Row Contention
Multiple updates on same row.

Fix:
- sharding by entity  
- append-only  
- optimistic locking  

## 12.2 Slow Queries
Fix:
- Add indexes  
- Use covering index  
- Avoid OR  
- Avoid OFFSET  

---

# 13. HIGH THROUGHPUT ARCHITECTURE PATTERNS

- async + batching  
- event-driven  
- CQRS  
- partitioning  
- write-ahead logs  
- append-only models  
- caching everywhere  
- outbox + Kafka  
- idempotent processors  
- hedging requests  

---

# 14. REAL-WORLD CASE STUDIES

### Uber
- pipelined Redis queries  
- Kafka batch processing  
- CPU cache-aware structures  

### Netflix
- aggressive async  
- tuned thread pools  
- use of Hystrix (circuit breakers)  

### DoorDash
- optimized Postgres indexes  
- tuned p99 latency  
- async order workflows  

---

# END OF CHAPTER I
