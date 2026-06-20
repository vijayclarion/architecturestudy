# One-Page Concept Summaries

Each concept on a single page. Print these, stick on wall, review daily.

---

## CACHE-ASIDE PATTERN

**What:** App manages cache. Read: check cache → miss → fetch DB → populate cache  
**When:** Read-heavy workloads (catalogs, profiles)  
**Why:** Memory efficient (only hot data cached)  
**Trade-off:** First request slow, stale data risk  
**Real:** Facebook Memcached, Netflix metadata  
**Soundbite:** *"Cache-Aside for read-heavy; cold start and staleness are accepted trade-offs"*

---

## CACHE STAMPEDE

**Problem:** Cache expires → 1000s of simultaneous DB queries → DB crash  
**Solutions:**
1. Mutex lock (one rebuilds, others wait)
2. Probabilistic early expiry (refresh before TTL)
3. Jittered TTL (random offset prevents sync expiry)
4. Stale-while-revalidate (serve stale, refresh background)  

**Real:** Facebook leases, Instagram TTL jitter  
**Soundbite:** *"Prevent stampede with mutex for single hot keys, jitter for distributed expiry"*

---

## CDN STRATEGY

**Headers:**
- `Cache-Control: public, max-age=86400, s-maxage=604800, immutable`
- `ETag` / `If-None-Match` for revalidation
- `stale-while-revalidate=60` for zero-latency refresh

**Versioned URLs > Purge API:**
- `/app.a1b2c3.js` — HTML always fresh, assets cached forever
- New version = new URL = no invalidation needed

**Real:** Cloudflare, Netflix Open Connect  
**Soundbite:** *"Long TTL + versioned URLs for static; short TTL + revalidation for dynamic"*

---

## CACHE INVALIDATION (EVENT-DRIVEN)

**Flow:**
```
DB change → Debezium (CDC) → Kafka/RabbitMQ → Consumers → Redis + CDN
```

**Patterns:**
- **CDC (Debezium):** Raw DB changes, no app code change
- **Outbox:** Atomic event in same transaction, business-semantic
- **Idempotent consumers:** Same event reprocessing is safe
- **DLQ:** Failed messages for retry/alert

**Real:** LinkedIn Databus, Shopify Debezium  
**Soundbite:** *"Event-driven via CDC for multi-layer invalidation; outbox when business semantics matter"*

---

## SQL vs NoSQL FUNDAMENTAL

| | SQL | Cassandra |
|---|---|---|
| Structure | B-Tree | LSM Tree |
| Write | Random I/O | Sequential |
| Read | Fast (indexed) | Fast (with partition key) |
| Scale | Vertical | Horizontal |
| CAP | CA / CP | AP |
| JOINs | Native | None |
| ACID | Full | Tunable |

**Decision:** Writes > 50k/sec → Cassandra. Need JOINs/ACID → SQL.  
**Soundbite:** *"SQL is storage-first, query-flexible. Cassandra is query-first, scale-first."*

---

## LSM TREE WRITE PATH

```
Write → Commit Log (disk, sequential) → MemTable (RAM)
                                              ↓ (when full)
                                         SSTable (immutable, disk)
                                              ↓
                                    Compaction (merge SSTables)
```

**Why fast:** All writes sequential, no random I/O, no locks  
**Why reads slower:** Data may span multiple SSTables → merge needed  
**Bloom filters:** Skip SSTables that don't have the key  

**Soundbite:** *"LSM trees turn writes into sequential disk operations, sacrificing read complexity for write throughput"*

---

## CONSISTENT HASHING

**Problem:** `hash(key) % N` — adding node remaps 75%+ of data  

**Solution:** Hash ring
- Nodes and keys on a circular ring (0° to 360°)
- Keys travel clockwise to find their node
- Adding a node → only adjacent data moves

**Virtual Nodes (vnodes):**
- 256 small ranges per physical node
- Even distribution + faster rebalancing

**Real:** DynamoDB, Cassandra, Redis Cluster, Memcached  
**Soundbite:** *"Consistent hashing minimizes data movement; vnodes ensure even distribution across heterogeneous nodes"*

---

## CASSANDRA PARTITIONING

```sql
PRIMARY KEY (user_id, order_time)
--          ▲ partition key (which node)
--                   ▲ clustering key (sort within partition)
```

**Rules:**
- Partition key determines node via consistent hash
- Always query by partition key (no full scans)
- Design tables per query (denormalize freely)
- Same data may live in multiple tables

**Anti-patterns:**
- Low cardinality partition key
- ALLOW FILTERING in production
- Secondary indexes on high-cardinality columns
- Partitions > 100MB

**Soundbite:** *"Query-first modeling: each query gets its own table, partition key drives distribution"*

---

## SQL PARTITIONING vs CASSANDRA PARTITIONING

| | SQL | Cassandra |
|---|---|---|
| Purpose | Performance opt within node | Data distribution across nodes |
| Where | Same machine | Different machines |
| Rebalancing | Manual | Automatic (gossip) |
| Cross-partition queries | JOINs work | Anti-pattern |
| Adding capacity | Bigger server | Add node |

**Soundbite:** *"SQL partitioning is a single-node optimization. Cassandra partitioning is the fundamental distribution mechanism."*

---

## GOSSIP PROTOCOL

**How:** Every node, every second, picks 1-3 random neighbors and shares state.

**What's shared:**
- Node status (UP/DOWN/SUSPECTED)
- Token assignments (ring position)
- Schema version
- Load metrics

**Failure detection:**
- Phi Accrual algorithm (probabilistic, adapts to network)
- Suspected → Down → Removed states

**Why no master:** No single point of failure, scales to 1000s of nodes  

**Real:** Cassandra, Consul, Riak  
**Soundbite:** *"Gossip protocol enables decentralized cluster awareness — no master, no SPOF, scales horizontally"*

---

## POLYGLOT PERSISTENCE

**Real-world pattern: use multiple databases**

| Data Type | Storage |
|---|---|
| Transactional | PostgreSQL |
| Time-series | Cassandra |
| Cache/session | Redis |
| Full-text search | Elasticsearch |
| Files/blobs | S3 |
| Graph | Neo4j |

**Why:** No single DB optimal for all workloads  
**Cost:** Operational complexity, data consistency between stores  

**Real:** Uber, Netflix, Zomato, Swiggy  
**Soundbite:** *"Polyglot persistence — pick the right tool per data shape; accept operational complexity for performance"*

---

## CAP THEOREM (PICK 2 OF 3)

- **C** Consistency: every read sees latest write
- **A** Availability: every request gets response (maybe stale)
- **P** Partition Tolerance: works despite network splits

**Reality:** P is mandatory in distributed systems → choose C or A

| System | Type |
|---|---|
| SQL (single node) | CA |
| SQL (distributed) | CP |
| Cassandra | AP |
| MongoDB | Tunable (CP default) |
| DynamoDB | Tunable |

**Soundbite:** *"CAP forces a choice between consistency and availability under network partition. Cassandra picks AP; most SQL picks CP."*

---

## QUICK INTERVIEW FRAMEWORK

For any system design question:

```
1. CLARIFY
   - Scale? Read/write ratio? Consistency needs?

2. SKETCH HIGH-LEVEL
   - Client → CDN → LB → App → Cache → DB

3. DIVE INTO COMPONENTS
   - Pick the most interesting/risky part

4. ARTICULATE TRADE-OFFS
   - "X gives us Y but costs Z"

5. PLAN FOR FAILURE
   - What if cache dies? DB partition?

6. CALL OUT THE LIMITS
   - "This works up to N. Beyond that, we'd need M."
```
