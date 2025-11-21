# E. Databases & Query Optimization (Principal Engineer Depth)

This chapter includes:

- PostgreSQL internals (MVCC, WAL, VACUUM, bloat)
- Query planner & execution pipeline
- Join algorithms (hash join, merge join, nested loop)
- Index types & when to use which
- Table partitioning & sharding strategies
- Redis internals (eviction, memory layout, latency behaviors)
- Caching patterns at scale
- Cassandra & LSM-tree internals
- Query optimization patterns
- DB anti-patterns
- High throughput DB architecture

---

# 1. POSTGRESQL INTERNALS (VERY DEEP)

PostgreSQL is not “just a database.”  
It is an **MVCC database with WAL-based durability, shared buffers, background writers, and vacuum workers**.

---

# 1.1 MVCC (Multi-Version Concurrency Control)

Instead of locks, Postgres stores:

- Tuple (row)  
- xmin (transaction that created it)  
- xmax (transaction that deleted/updated it)

### Why MVCC?
- Readers never block writers  
- Writers never block readers  
- Isolation without locking everything  

---

## 1.2 Tuple Lifecycle Diagram

```
INSERT:
  tuple1 (xmin=T1, xmax=null)

UPDATE:
  tuple1: xmax=T2    ← old row (dead)
  tuple2: xmin=T2    ← new row (live)

DELETE:
  tupleX: xmax=T3    ← dead row
```

Dead rows accumulate → VACUUM must clean.

---

# 1.3 VACUUM (Deep Explanation)

VACUUM:
- Removes dead tuples
- Frees space (if HOT)
- Prevents table bloat
- Updates visibility maps

HOT Updates:
- HOT = “Heap Only Tuple”
- Happens when update doesn't change indexed fields
- Faster updates, less index churn

---

# 1.4 Table Bloat

Because Postgres never overwrites rows:

- UPDATE → creates new row  
- DELETE → marks old row dead  

If VACUUM is not aggressive enough → table grows forever.

Detect bloat:

```sql
SELECT * FROM pgstattuple('users');
```

---

# 1.5 WAL (Write-Ahead Log)

Before writing to data files:

```
Write log entry → fsync WAL → write to heap later
```

Guarantees:
- crash safety  
- durability  

WAL is append-only → high throughput.

---

# 1.6 Shared Buffers & OS Page Cache

Flow:

```
Query
  ↓
Shared Buffer (Postgres memory)
  ↓ miss
OS Page Cache
  ↓ miss
Disk read
```

DB tuning = managing buffer hit ratios.

---

# 2. QUERY PLANNER INTERNALLY

Postgres planner builds a plan tree:

```
Query
  ↓ parse
Rewrite rules (views, RLS)
  ↓
Planner (cost-based)
  ↓
Executor
```

Planner decides:
- join order  
- join algorithm  
- index usage  
- parallelism  
- filter pushdown  

---

## 2.1 Explain Analyze (Example)

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
```

Output fields:
- Actual Rows  
- Planning Time  
- Execution Time  
- Buffers  
- I/O statistics  

---

# 2.2 Cost Model (Important!)

Postgres uses a cost model:

```
Cost = CPU cost + I/O cost
```

It **does not know actual row count**, only estimates.

Problems:
- Outdated statistics → bad plans  
- Skewed data → bad estimates  

Fix:
```
ANALYZE orders;
```

---

# 3. JOIN ALGORITHMS (DEEP)

## 3.1 Nested Loop Join

```
for row in A:
    for row in B:
        join
```

Best when:
- small inner table  
- indexes on lookup keys  

Worst when:
- join both large tables  

---

## 3.2 Hash Join

Builds hash table on smaller table.

```
Build hash(B)
for row in A:
   check hash(B)
```

Best when:
- equality join  
- large tables  
- no indexes  

---

## 3.3 Merge Join

Requires sorted inputs.

```
scan A sorted
scan B sorted
merge
```

Best when:
- both sides sorted  
- indexed merge  

---

# 4. INDEXES (ULTRA-DEEP SECTION)

Indexes are not free — they:

- speed reads  
- slow writes  
- increase storage  
- impact VACUUM  

---

## 4.1 B-Tree (default)

Great for:
- equality  
- range queries  
- sorting  

Bad for:
- full-text search  
- array overlap  

---

## 4.2 GIN Index

For:
- full-text search  
- JSONB  
- array containment (`@>`)  

GIN is expensive to update.

---

## 4.3 GiST Index

For:
- geo-spatial  
- kNN  
- trigrams  

---

## 4.4 BRIN Index

Block Range Index.

Shines when:
- data is append-only
- values correlate with insertion order  
- huge tables (TB-scale)

Example:
- timestamp logs

---

# 4.5 Partial Indexes (Most Underused Feature)

```sql
CREATE INDEX active_users_idx ON users(id) WHERE active = true;
```

Benefits:
- smaller index  
- faster queries  
- lower maintenance  

---

# 4.6 Covering Index (INCLUDE)

```sql
CREATE INDEX idx_x ON table(a,b) INCLUDE(c,d);
```

Allows index-only scans.

---

# 4.7 Index Anti-Patterns

❌ Too many indexes  
❌ Indexing low-selectivity columns  
❌ Indexing booleans  
❌ Indexing huge JSON blobs  
❌ Forgetting to ANALYZE after bulk insert  

---

# 5. SHARDING & PARTITIONING

## 5.1 Partitioning Types

### Range
```
WHERE created_at BETWEEN ...
```

### Hash
```
MOD(user_id, N)
```

### List
Categorical splitting (countries, regions).

---

## 5.2 Sharding Strategies

### User-based sharding
```
shard = user_id % N
```

Pros:
- predictable  
- low cross-shard joins  

### Time-based sharding
For logs, events.

### Geo-sharding  
Proximity-based.

---

# 6. REDIS INTERNALS

Redis is **single-threaded**, but extremely fast.

---

## 6.1 Redis as In-Memory Hash Tables

Redis uses:
- ziplist → small
- hashtable → bigger
- quicklist for lists
- skiplist for sorted sets

---

## 6.2 Eviction Strategies

```
volatile-lru
allkeys-lru
volatile-lfu
allkeys-lfu
volatile-ttl
noeviction
```

LRU != perfect LRU  
Redis uses approximated LRU.

---

## 6.3 Redis Latency Spikes

Causes:
- eviction  
- RDB/AOF rewrite  
- huge keys  
- slow clients  
- cluster resharding  
- fork during snapshot  

---

## 6.4 Redis Pipelines

Batch operations reduce round-trip latency:

```
MULTI
SET key1 val1
SET key2 val2
EXEC
```

---

# 7. CASSANDRA & LSM-TREE INTERNALS

Cassandra uses **Log Structured Merge Trees**.

---

## 7.1 LSM Tree Structure

```
writes → memtable (in RAM)
          ↓ flush
        SSTables (immutable, disk)
```

Reads:
- merge SSTables + memtable  
- bloom filter check  
- compaction removes outdated rows  

---

## 7.2 Compaction

- Merges SSTables  
- Removes tombstones  
- Reduces read amplification  

---

## 7.3 When to use Cassandra

- massive writes  
- append-only workloads  
- global scale  
- eventual consistency acceptable  

---

# 8. CACHING AT SCALE (DEEP)

Cache types:

- Local in-memory (Caffeine)
- Redis
- CDN
- Application-level

---

## 8.1 Cache Invalidation Patterns

```
Cache-aside
Write-through
Write-behind
Refresh-ahead
```

---

## 8.2 Cache Stampede

Too many requests for missing key.

Solutions:
- mutex lock  
- stale-while-revalidate  
- jittered TTL  

---

# 9. QUERY OPTIMIZATION PLAYBOOK

## 9.1 Avoid SELECT *

Use explicit columns.

## 9.2 Avoid OR

Rewrite with UNION or index-friendly operations.

## 9.3 Use LIMIT wisely

Avoid large offset:

Bad:
```sql
LIMIT 50 OFFSET 100000
```

Better:
```sql
WHERE id > last_seen_id LIMIT 50
```

## 9.4 Avoid functions on indexed column

Bad:
```sql
WHERE date(created_at) = '2025-01-01'
```

Better:
```sql
WHERE created_at >= '2025-01-01' AND created_at < '2025-01-02'
```

---

# 10. DB ANTI-PATTERNS

❌ 1. Storing large blobs in DB  
❌ 2. Using SERIAL instead of UUID in distributed systems  
❌ 3. Joins across microservices  
❌ 4. Unbounded table growth  
❌ 5. Storing JSON everywhere  
❌ 6. Entity-based microservice boundaries  
❌ 7. Un-indexed foreign keys  
❌ 8. Not using batch inserts  
❌ 9. Using ORM for everything  

---

# 11. HIGH THROUGHPUT DATABASE DESIGN

## Principles:

- Keep hot rows independent  
- Partition by access pattern  
- Use append-only tables  
- Split read/write paths  
- Use outbox  
- Cache aggressively  
- Use bulk operations  

---

# END OF CHAPTER E
