# Cache-Through Patterns (Write-Through, Read-Through, Write-Behind, Write-Around)

## Problem It Solves

Cache-Aside puts all caching logic in the application. As systems grow, this becomes:
- **Repetitive** — every service writes the same cache logic
- **Error-prone** — easy to forget invalidation
- **Inconsistent** — different services handle caching differently
- **Stale data risk** — gap between DB write and cache invalidation

**Cache-Through patterns** make the cache itself responsible for keeping in sync with the DB.

---

## The Four Patterns at a Glance

```mermaid
graph TB
    subgraph "1. Cache-Aside (app manages)"
        A1[App] -->|read| C1[Cache]
        A1 -->|on miss| D1[(DB)]
        A1 -->|write| D1
        A1 -.invalidate.-> C1
    end
    
    subgraph "2. Read-Through (cache reads DB)"
        A2[App] -->|read only| C2[Cache]
        C2 -->|on miss| D2[(DB)]
        A2 -->|write| D2
    end
    
    subgraph "3. Write-Through (cache writes DB sync)"
        A3[App] -->|read/write| C3[Cache]
        C3 -->|sync write| D3[(DB)]
    end
    
    subgraph "4. Write-Behind (cache writes DB async)"
        A4[App] -->|read/write| C4[Cache]
        C4 -.async.-> D4[(DB)]
    end
    
    style C1 fill:#ffe1e1
    style C2 fill:#e1f5ff
    style C3 fill:#ccffcc
    style C4 fill:#fff4e1
```

---

## Pattern 1 — Write-Through Cache

### How It Works

```mermaid
sequenceDiagram
    autonumber
    participant App as Application
    participant Cache
    participant DB as Database

    rect rgb(220, 255, 220)
        Note over App,DB: WRITE flow
        App->>Cache: PUT product:123 (data)
        Cache->>DB: INSERT/UPDATE products
        DB-->>Cache: ACK
        Cache-->>App: SUCCESS
        Note over Cache: Cache + DB always in sync
    end
    
    rect rgb(220, 220, 255)
        Note over App,DB: READ flow
        App->>Cache: GET product:123
        Cache-->>App: Return cached data
        Note over Cache: Always serves from cache
    end
```

### Key Characteristics

- **Write path:** App → Cache → DB (synchronous)
- **Read path:** App → Cache (always hits cache)
- **Consistency:** Strong — cache and DB always match
- **Latency:** Higher writes (double hop), low reads
- **App complexity:** Low — no cache management logic

### When to Use

- ✅ **Read-heavy with critical consistency** (user profiles, settings)
- ✅ **Financial data** — cannot tolerate stale reads
- ✅ **Multiple services** sharing same cache (consistent view)
- ✅ When **read amplification** would benefit from always-cached data

### When NOT to Use

- ❌ **Write-heavy workloads** — every write pays double latency
- ❌ **Rarely-read data** — cache pollution (caches everything written)
- ❌ **Limited cache memory** — too much data, low hit ratio

### Trade-offs

| Gain | Sacrifice |
|---|---|
| Strong consistency | Higher write latency |
| Simpler app code | Cache pollution (every write cached) |
| No stale data risk | Cache failure blocks writes |
| Always-warm cache | Wasted memory on never-read keys |

### Real World Example

**DynamoDB Accelerator (DAX)** uses write-through — writes go to DAX which synchronously persists to DynamoDB. Reads served at microsecond latency.

**Microsoft Azure Cache for Redis** offers write-through integration with Azure SQL.

---

## Pattern 2 — Read-Through Cache

### How It Works

```mermaid
sequenceDiagram
    autonumber
    participant App as Application
    participant Cache
    participant DB as Database

    rect rgb(220, 220, 255)
        Note over App,DB: READ flow (cache hit)
        App->>Cache: GET product:123
        Cache-->>App: Return data
    end
    
    rect rgb(255, 220, 220)
        Note over App,DB: READ flow (cache miss)
        App->>Cache: GET product:123
        Cache->>DB: SELECT (cache fetches!)
        DB-->>Cache: Data
        Cache->>Cache: Store locally
        Cache-->>App: Return data
    end
    
    rect rgb(220, 255, 220)
        Note over App,DB: WRITE flow (app writes DB directly)
        App->>DB: UPDATE products
        DB-->>App: ACK
        Note over Cache: Cache invalidated separately<br/>(via TTL or event)
    end
```

### Key Characteristics

- **Read path:** App → Cache (cache fetches from DB on miss)
- **Write path:** App → DB directly (cache not involved)
- **App complexity:** Very low — treats cache like a DB
- **Cache integration:** Cache library/proxy handles DB fetch

### Cache-Aside vs Read-Through — Crucial Difference

```mermaid
graph LR
    subgraph "Cache-Aside (App does work)"
        A1[App Code] -->|"1. GET"| C1[Cache]
        C1 -.miss.-> A1
        A1 -->|"2. SELECT"| D1[(DB)]
        A1 -->|"3. SET"| C1
    end
    
    subgraph "Read-Through (Cache does work)"
        A2[App Code] -->|"1. GET"| C2[Cache]
        C2 -->|"2. SELECT (internal)"| D2[(DB)]
        C2 -.return.-> A2
    end
    
    style A1 fill:#ffcccc
    style C2 fill:#ccffcc
```

In Cache-Aside, **app code** handles the miss logic. In Read-Through, **cache library** handles it transparently.

### When to Use

- ✅ Want to **simplify app code** (looks like direct DB access)
- ✅ Multiple services need same caching behavior
- ✅ Want **consistent miss handling** across the system
- ✅ Using a cache framework that supports it (Spring Cache, Hibernate, EhCache)

### Trade-offs

| Gain | Sacrifice |
|---|---|
| Simple app code | First read always slow (cold cache) |
| Consistent miss logic | Less flexibility per-query |
| Easy to add caching | Requires cache that supports it |
| Centralized cache logic | Harder to debug |

### Real World Example

**Hibernate Second-Level Cache** is read-through — Hibernate transparently fetches from DB on cache miss.

**Spring Cache with @Cacheable** annotation — declarative read-through caching with one line of code.

---

## Pattern 3 — Write-Behind (Write-Back) Cache

### How It Works

```mermaid
sequenceDiagram
    autonumber
    participant App as Application
    participant Cache
    participant Q as Write Queue
    participant DB as Database

    rect rgb(220, 255, 220)
        Note over App,DB: WRITE flow (super fast)
        App->>Cache: PUT product:123
        Cache->>Q: Queue write
        Cache-->>App: SUCCESS (immediate!)
    end
    
    rect rgb(255, 240, 220)
        Note over App,DB: ASYNC flush to DB
        Q-->>DB: Batch write (every Xs or N writes)
        DB-->>Q: ACK
    end
    
    rect rgb(220, 220, 255)
        Note over App,DB: READ flow
        App->>Cache: GET product:123
        Cache-->>App: Return (from cache)
    end
```

### Key Characteristics

- **Write path:** App → Cache → (async) → DB
- **Write latency:** Extremely low (microseconds)
- **Throughput:** Very high — writes batched
- **Risk:** Data loss if cache fails before DB write
- **Consistency:** Eventual

### Batching Strategy

```mermaid
graph TB
    W1[Write 1] --> Q[Queue/Buffer]
    W2[Write 2] --> Q
    W3[Write 3] --> Q
    Wn[Write N] --> Q
    
    Q -->|Trigger conditions| B[Batch Flush]
    
    T1[Every 5 seconds] -.trigger.-> B
    T2[Every 1000 writes] -.trigger.-> B
    T3[Cache memory pressure] -.trigger.-> B
    
    B --> DB[(Database)]
    
    style Q fill:#fff4e1
    style B fill:#ccffcc
```

### When to Use

- ✅ **Write-heavy workloads** (analytics, logging, metrics)
- ✅ **Burst tolerance** — smooth out write spikes to DB
- ✅ **DB throughput limited** — batch reduces DB load
- ✅ **Latency-critical writes** — gaming leaderboards, view counts
- ✅ Acceptable to **lose a few writes** on cache failure

### When NOT to Use

- ❌ **Financial transactions** — cannot lose any writes
- ❌ **Strong consistency required**
- ❌ **Single-tenant cache** — high blast radius on failure

### Trade-offs

| Gain | Sacrifice |
|---|---|
| Ultra-fast writes | Risk of data loss |
| High throughput | Eventual consistency |
| DB protected from spikes | Complex failure handling |
| Batch write efficiency | Hard to debug write delays |

### Resilience Techniques

```mermaid
graph TB
    App[App Write] --> C[Cache]
    C --> WAL[Write-Ahead Log on disk]
    C --> Q[Memory Queue]
    Q --> DB[(Database)]
    
    WAL -.recover after crash.-> Q
    
    style WAL fill:#ffcccc
    style Q fill:#fff4e1
```

To minimize data loss:
- **Write-ahead log (WAL)** — persist queue to disk first
- **Replication** — replicate cache state across nodes
- **Acknowledged batches** — only remove from queue after DB ACK

### Real World Example

**Redis with AOF (Append-Only File)** can implement write-behind by buffering writes and flushing to DB asynchronously.

**Linux page cache** is write-behind by default — `write()` returns immediately, OS flushes to disk later.

**Apache Ignite, Hazelcast** offer built-in write-behind with configurable batching.

---

## Pattern 4 — Write-Around Cache

### How It Works

```mermaid
sequenceDiagram
    autonumber
    participant App as Application
    participant Cache
    participant DB as Database

    rect rgb(220, 255, 220)
        Note over App,DB: WRITE flow - skips cache
        App->>DB: UPDATE products
        DB-->>App: ACK
        Note over Cache: Cache NOT touched on write
    end
    
    rect rgb(255, 220, 220)
        Note over App,DB: READ flow (cache miss on first read)
        App->>Cache: GET product:123
        Cache-->>App: MISS
        App->>DB: SELECT
        DB-->>App: Data
        App->>Cache: SET (now cached)
    end
    
    rect rgb(220, 220, 255)
        Note over App,DB: Subsequent reads hit cache
        App->>Cache: GET product:123
        Cache-->>App: HIT
    end
```

### Key Characteristics

- **Write path:** App → DB directly (cache bypassed)
- **Read path:** Cache-Aside style (populate on miss)
- **Cache contains:** Only frequently READ data, not all writes
- **Memory efficient:** Doesn't cache write-only data

### When to Use

- ✅ **Write-once, read-rarely** data (audit logs, archives)
- ✅ **Bulk imports** — don't pollute cache with rarely-read rows
- ✅ Combined with Cache-Aside or Read-Through for reads
- ✅ **Limited cache memory** — only cache hot data

### Trade-offs

| Gain | Sacrifice |
|---|---|
| Cache has only hot data | First read always slow |
| No write latency overhead | Possible stale data after write |
| Memory efficient | Need TTL/eventing for invalidation |

### Real World Example

**Audit log systems** — every action writes to DB but only recent entries are read; write-around prevents flooding cache with old logs.

**Bulk data import jobs** — load millions of rows to DB without polluting cache.

---

## Full Comparison Matrix

| Pattern | Write Path | Read Path | Write Latency | Read Latency | Consistency | Best For |
|---|---|---|---|---|---|---|
| **Cache-Aside** | App → DB; invalidate cache | App → Cache; on miss App → DB → Cache | Low | High on miss, Low on hit | Eventual | Read-heavy, flexible |
| **Read-Through** | App → DB; invalidate cache | App → Cache; on miss Cache → DB | Low | High on miss, Low on hit | Eventual | Simple app code |
| **Write-Through** | App → Cache → DB (sync) | App → Cache | High (double-hop) | Low | Strong | Critical consistency |
| **Write-Behind** | App → Cache → async DB | App → Cache | Very Low | Low | Eventual | Write-heavy |
| **Write-Around** | App → DB directly | App → Cache; on miss App → DB | Low | High on miss | Eventual | Write-once read-rarely |

---

## Decision Tree

```mermaid
graph TB
    Q[What's your workload?] --> R{Read or Write heavy?}
    
    R -->|Read-heavy| RH{Consistency critical?}
    R -->|Write-heavy| WH{Can tolerate data loss?}
    R -->|Balanced| BAL{Want simple code?}
    
    RH -->|Yes| WT[Write-Through]
    RH -->|No| CA[Cache-Aside]
    
    WH -->|Yes| WB[Write-Behind]
    WH -->|No| WT2[Write-Through + Replication]
    
    BAL -->|Yes| RT[Read-Through]
    BAL -->|No| CA2[Cache-Aside]
    
    WO[Write-once, read-rarely?] --> WA[Write-Around]
    
    style WT fill:#ccffcc
    style WB fill:#fff4e1
    style CA fill:#ffe1e1
    style RT fill:#e1f5ff
    style WA fill:#f0e1ff
```

---

## Combining Patterns — Real Architectures

Most real systems use **multiple patterns** for different data:

```mermaid
graph TB
    APP[E-commerce App]
    
    APP -->|User profiles| WT[Write-Through Cache]
    APP -->|Product catalog| CA[Cache-Aside]
    APP -->|View counts| WB[Write-Behind]
    APP -->|Order history| WA[Write-Around]
    APP -->|Session data| RT[Read-Through]
    
    WT --> DB1[(User DB)]
    CA --> DB2[(Product DB)]
    WB --> DB3[(Analytics DB)]
    WA --> DB4[(Orders DB)]
    RT --> DB5[(Session Store)]
    
    style WT fill:#ccffcc
    style CA fill:#ffe1e1
    style WB fill:#fff4e1
    style WA fill:#f0e1ff
    style RT fill:#e1f5ff
```

---

## Common Mistakes

❌ **Using Write-Through for write-heavy workload** — every write pays double latency  
❌ **Using Write-Behind for financial data** — data loss risk  
❌ **Mixing patterns inconsistently** — same data with different patterns causes chaos  
❌ **Ignoring cache failure scenarios** — what happens when cache is down?  
❌ **Not measuring hit ratio** — without metrics, you don't know if caching helps  

---

## Interview Soundbite

> *"I don't pick one cache pattern globally — I pick per data type. Write-Through for critical reads needing consistency, like user profiles. Cache-Aside for product catalog where staleness is OK. Write-Behind for high-volume writes like view counters where eventual consistency is fine. Write-Around for write-once-read-rarely data like audit logs. The key trade-off in all of these is consistency vs latency vs operational complexity."*

---

## Interview Follow-up Questions to Practice

1. *"In Write-Through, the DB write fails after cache write succeeds. What happens?"*
2. *"How do you implement Write-Behind without losing data on cache crash?"*
3. *"Read-Through requires cache to access DB — isn't that a tight coupling?"*
4. *"Cache-Aside has stale data window between DB write and cache invalidation. How do you reduce it?"*
5. *"You have Write-Through cache. Cache cluster goes down. Do writes block or fail-fast?"*
