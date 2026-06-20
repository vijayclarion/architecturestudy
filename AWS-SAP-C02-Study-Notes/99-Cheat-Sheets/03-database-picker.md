# Database Decision Tree — Cheat Sheet

> 🔥 **Recurring SAP-C02 question: "Which database service?"** Walk this tree to win every time.

---

## 🌳 The Master Decision Tree

```mermaid
flowchart TD
    START[Need a database?]
    START --> Q1{Data model?}
    
    Q1 -->|Relational, ACID| REL[See Relational Branch]
    Q1 -->|Key-value| KV[See Key-Value Branch]
    Q1 -->|Document JSON| DOC[See Document Branch]
    Q1 -->|Graph relationships| NEP[Amazon Neptune]
    Q1 -->|Wide column - Cassandra| KEY[Keyspaces]
    Q1 -->|Time series| TS[Timestream]
    Q1 -->|Immutable ledger| QLDB[QLDB]
    Q1 -->|Analytics / OLAP| OLAP[Redshift / Athena]
    Q1 -->|In-memory cache| CACHE[ElastiCache / MemoryDB]
    Q1 -->|Search / Logs| SEARCH[OpenSearch]
    Q1 -->|Vector / AI| VEC[See Vector Branch]
```

---

## 🗄️ Relational Branch

```mermaid
flowchart TD
    REL[Relational Need] --> Q1{Multi-region writes?}
    
    Q1 -->|Yes - active-active needed| DDB[Cannot use SQL natively<br/>Reconsider — use DynamoDB GT]
    Q1 -->|No - single primary OK| Q2{Engine compatibility?}
    
    Q2 -->|Oracle, SQL Server| RDS_COMM[RDS for that engine]
    Q2 -->|MySQL, PostgreSQL, MariaDB| Q3{Performance critical?}
    
    Q3 -->|Yes - cloud-native scale| AURORA[Aurora]
    Q3 -->|Simple workload, cost-sensitive| RDS[RDS for MySQL/PG]
    
    AURORA --> Q4{Workload pattern?}
    Q4 -->|Variable, dev/test, SaaS| AS[Aurora Serverless v2]
    Q4 -->|Multi-region reads + DR| AG[Aurora Global Database]
    Q4 -->|Steady production| AP[Aurora Provisioned]
```

```
QUICK PICK:
─────────────────────────────────────────────
Need Oracle features         → RDS Oracle
Need SQL Server              → RDS SQL Server
Migrating off Oracle         → Aurora PostgreSQL (with Babelfish)
                               or Aurora MySQL
Modern PG/MySQL              → Aurora
Variable workload            → Aurora Serverless v2
Multi-region active-passive  → Aurora Global Database
Simple, cheap                → RDS for MySQL or PG
On-prem migration            → RDS same engine first, optimize later
```

---

## 🔑 Key-Value Branch

```mermaid
flowchart TD
    KV[Key-Value Need] --> Q1{Scale?}
    
    Q1 -->|Massive - 1000s/sec+| Q2{Multi-region?}
    Q1 -->|Modest, want managed| DDB1[DynamoDB On-Demand]
    
    Q2 -->|Yes - active-active| DGT[DynamoDB Global Tables]
    Q2 -->|No - single region| Q3{Latency req?}
    
    Q3 -->|< 1ms required| DAX[DynamoDB + DAX]
    Q3 -->|Single digit ms OK| DDB2[DynamoDB]
    
    DDB1 --> Q4{Predictable load?}
    Q4 -->|Yes| DDB_PROV[Provisioned + Auto Scaling]
    Q4 -->|Spiky/unknown| DDB_OD[On-Demand]
```

```
QUICK PICK:
─────────────────────────────────────────────
General serverless KV         → DynamoDB
Multi-region active-active    → DynamoDB Global Tables
Microsecond reads             → DynamoDB + DAX
Pure cache (no source needed) → ElastiCache Redis
Cache that must be durable    → MemoryDB for Redis
```

---

## 📄 Document Branch

```
DOCUMENT DATABASES:
─────────────────────────────────────────────
DynamoDB           → Schema-flexible, key-value or doc up to 400KB
                     ✅ AWS-native, cheap, scales
                     
DocumentDB         → MongoDB API compatible
                     ✅ For workloads expecting MongoDB
                     ⚠️ Not 100% MongoDB feature parity
                     
DynamoDB vs DocumentDB:
─────────────────────────────────────────────
Use DynamoDB if greenfield (better, cheaper, more scalable)
Use DocumentDB if you must keep MongoDB compatibility
```

---

## 🔍 Specialized Database Branch

```mermaid
flowchart TD
    SPEC[Specialized Need] --> N1[Graph]
    SPEC --> N2[Time-Series]
    SPEC --> N3[Ledger/Immutable]
    SPEC --> N4[Wide-Column]
    SPEC --> N5[Vector/Embeddings]
    
    N1 --> NEPTUNE[Amazon Neptune<br/>Property Graph + RDF/SPARQL]
    N2 --> TS[Amazon Timestream<br/>Native time-series]
    N3 --> QLDB[Amazon QLDB<br/>Cryptographic ledger]
    N4 --> KEY[Keyspaces<br/>Cassandra-compatible]
    N5 --> VEC[Aurora PostgreSQL + pgvector<br/>or OpenSearch k-NN]
```

### Specialized DBs At a Glance

| Database | Best For | Real-World Use |
|---|---|---|
| **Neptune** | Relationships, traversals | Fraud detection, recommendation engines, knowledge graphs |
| **Timestream** | Time-stamped data | IoT telemetry, app monitoring, industrial sensors |
| **QLDB** | Immutable history | Financial ledgers, supply chain, regulatory audit |
| **Keyspaces** | Cassandra workloads | High-throughput writes, IoT, gaming leaderboards |
| **Vector (Aurora/OpenSearch)** | Embeddings, semantic search | RAG for LLMs, image similarity, recommendations |

---

## 📊 OLTP vs OLAP

```
OLTP — Online Transaction Processing:
─────────────────────────────────────────────
- Many small transactions
- Read AND write heavy
- Latency-sensitive
- Examples: e-commerce, banking
- Services: RDS, Aurora, DynamoDB

OLAP — Online Analytical Processing:
─────────────────────────────────────────────
- Few large queries scanning lots of data
- Read-heavy (writes are batch loads)
- Complex aggregations, joins
- Examples: BI dashboards, data warehouses
- Services: Redshift, Athena, EMR
```

---

## 📈 Analytics Branch

```mermaid
flowchart TD
    ANA[Analytics Need] --> Q1{Workload type?}
    
    Q1 -->|Pre-loaded warehouse| RED[Redshift]
    Q1 -->|Ad-hoc SQL on S3| ATH[Athena]
    Q1 -->|Big batch ETL| EMR[EMR Spark]
    Q1 -->|Real-time stream| KIN[Kinesis Analytics<br/>or Managed Flink]
    
    RED --> Q2{Variable workload?}
    Q2 -->|Yes| RS[Redshift Serverless]
    Q2 -->|Steady| RP[Redshift Provisioned]
```

---

## 🎯 Common SAP-C02 Scenarios

### Scenario 1: Multi-Region Active-Active OLTP

> "Globally distributed app, users write from any continent, eventual consistency OK."

✅ **DynamoDB Global Tables**
- True active-active, < 1 sec lag
- Aurora Global is active-passive — wrong

---

### Scenario 2: Migrating from Oracle, Want PostgreSQL

> "Off Oracle licensing, similar SQL functionality, multi-region DR."

✅ **Aurora PostgreSQL** + Aurora Global Database
- DMS + SCT to migrate
- Possibly **Babelfish** if app uses T-SQL features

---

### Scenario 3: Spiky SaaS Multi-Tenant

> "B2B SaaS, each tenant has its own DB, traffic varies by tenant."

✅ **Aurora Serverless v2**
- Sub-second scaling, can be 0.5 ACU when idle
- Per-tenant DB pattern works well

---

### Scenario 4: IoT Telemetry at Scale

> "Millions of devices sending readings every minute."

✅ **DynamoDB On-Demand** (PK = DeviceId) for low latency lookups
✅ **Timestream** for time-series analytics on the data
✅ Or **Keyspaces** if migrating from Cassandra

---

### Scenario 5: Recommendation Engine

> "Need to model relationships: users → products → similar products → similar users."

✅ **Amazon Neptune** (property graph)
- Cypher or Gremlin queries
- Built for relationship traversal

---

### Scenario 6: Banking Ledger

> "Need provable, immutable transaction history for audit."

✅ **Amazon QLDB**
- Cryptographically verifiable
- Append-only, never updated/deleted
- Compliance-friendly

---

### Scenario 7: Gaming Leaderboard

> "Show top 100 players globally, updated every second."

✅ **ElastiCache Redis** (Sorted Sets) backed by DynamoDB
- Sorted Set ZADD + ZRANGE — perfect fit
- Redis multi-AZ replication for HA

---

### Scenario 8: Embeddings / RAG for LLM

> "Storing 1M vector embeddings for semantic search."

✅ **Aurora PostgreSQL + pgvector extension**
- Or **OpenSearch** with k-NN
- Or **Aurora Serverless v2 + pgvector** for variable cost

---

### Scenario 9: Existing Cassandra Workload

> "Lift-and-shift Cassandra to managed service."

✅ **Amazon Keyspaces**
- CQL-compatible
- Serverless or provisioned modes

---

## ⚡ The Master Quick Recall Card

```
┌─────────────────────────────────────────────────┐
│ DATABASE PICKER ULTRA-QUICK                      │
├─────────────────────────────────────────────────┤
│ RELATIONAL                                       │
│   Oracle/SQLServer           → RDS               │
│   MySQL/PG modern            → Aurora            │
│   Variable workload          → Aurora Serverless │
│   Multi-region read/DR       → Aurora Global     │
├─────────────────────────────────────────────────┤
│ NoSQL                                            │
│   Key-Value at scale         → DynamoDB          │
│   DDB multi-region writes    → DDB Global Tables │
│   DDB microsecond reads      → DAX               │
│   MongoDB API                → DocumentDB        │
│   Cassandra API              → Keyspaces         │
├─────────────────────────────────────────────────┤
│ SPECIALIZED                                      │
│   Graph                      → Neptune           │
│   Time-series                → Timestream        │
│   Immutable ledger           → QLDB              │
│   Vector / embeddings        → Aurora pgvector   │
├─────────────────────────────────────────────────┤
│ ANALYTICS                                        │
│   Data warehouse             → Redshift          │
│   Ad-hoc SQL on S3           → Athena            │
│   Big Spark/Hadoop           → EMR               │
│   Real-time stream SQL       → Managed Flink     │
├─────────────────────────────────────────────────┤
│ CACHE / SPEED                                    │
│   General cache              → ElastiCache Redis │
│   Cache that's durable       → MemoryDB          │
│   Simple multi-thread cache  → ElastiCache Memcached│
├─────────────────────────────────────────────────┤
│ SEARCH                                           │
│   Full-text / log analytics  → OpenSearch        │
└─────────────────────────────────────────────────┘
```
