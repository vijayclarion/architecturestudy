# Architect Decision Trees

Quick reference for interview answers. Memorize these decision flows.

---

## 1. Caching Strategy Decision Tree

```
What are you caching?
│
├── Static assets (JS, CSS, images)?
│   └── CDN with long TTL + versioned URLs
│
├── User session data?
│   └── Redis with short TTL
│
├── Database query results (read-heavy)?
│   └── Cache-Aside (Redis) with TTL
│
├── Database writes (must stay consistent)?
│   └── Write-Through cache
│
├── Computed/expensive results?
│   └── Cache-Aside with longer TTL
│
└── Real-time data (stock prices, scores)?
    └── Don't cache OR very short TTL with stale-while-revalidate
```

---

## 2. Cache Invalidation Decision Tree

```
How is data updated?
│
├── Rarely (catalog metadata)?
│   └── TTL only (1 hour to 1 day)
│
├── Predictably (daily batch)?
│   └── Scheduled invalidation + TTL
│
├── User-triggered (price change)?
│   ├── Single cache layer?
│   │   └── Write-through invalidation
│   └── Multiple layers (Redis + CDN)?
│       └── Event-driven via CDC (Debezium → Kafka)
│
└── Real-time critical (inventory)?
    └── Versioned URLs OR skip cache entirely
```

---

## 3. SQL vs NoSQL Decision Tree

```
What are your requirements?
│
├── Need ACID transactions?
│   └── YES → SQL (PostgreSQL, MySQL)
│
├── Writes > 50k/sec sustained?
│   ├── YES → Cassandra
│   └── NO → continue
│
├── Need complex JOINs / reporting?
│   ├── YES → SQL
│   └── NO → continue
│
├── Multi-region active-active writes?
│   ├── YES → Cassandra
│   └── NO → continue
│
├── Document-shaped data with flexible schema?
│   └── MongoDB / DynamoDB
│
├── Time-series (metrics, logs, IoT)?
│   └── Cassandra OR TimescaleDB OR InfluxDB
│
├── Graph relationships (social, recommendations)?
│   └── Neo4j / Neptune
│
└── Default (moderate scale, complex queries)?
    └── PostgreSQL (modern Postgres handles huge scale)
```

---

## 4. Database Partitioning Decision Tree

```
Why do you need partitioning?
│
├── Performance optimization within one server?
│   └── SQL table partitioning (range/hash/list)
│
├── Single server can't handle write load?
│   ├── Can you redesign queries?
│   │   ├── YES → Move to Cassandra
│   │   └── NO → SQL sharding (painful)
│
├── Global distribution required?
│   └── Cassandra (multi-DC built in)
│
└── Archival / old data management?
    └── SQL range partitioning by date
```

---

## 5. Consistency Level Decision Tree (Cassandra)

```
What's the data criticality?
│
├── User-facing critical (account balance)?
│   └── R + W > N (QUORUM + QUORUM with RF=3)
│
├── Analytics / logs?
│   └── W=ONE (fast writes, OK with delay)
│
├── Multi-region?
│   └── LOCAL_QUORUM (avoid cross-region latency)
│
└── Read-heavy with eventual consistency OK?
    └── R=ONE (fast reads)
```

---

## 6. Cache Failure Handling Decision Tree

```
Cache went down. What happens?
│
├── App falls through to DB?
│   ├── DB can handle the load?
│   │   └── OK, but watch for stampede
│   └── DB will crash?
│       └── Circuit breaker — serve degraded
│
├── Use stale data instead?
│   └── stale-while-revalidate pattern
│
└── Reject requests gracefully?
    └── HTTP 503 with retry-after header
```

---

## 7. Message Broker Choice (for invalidation/events)

```
What's your requirement?
│
├── Event log / replay needed?
│   └── Kafka (durable log)
│
├── Task queue / work distribution?
│   └── RabbitMQ
│
├── Massive throughput, multi-consumer?
│   └── Kafka
│
├── Simple pub-sub, low ops overhead?
│   └── Redis Pub/Sub OR AWS SNS/SQS
│
└── Cloud-native with autoscale?
    └── AWS Kinesis / Google Pub-Sub
```

---

## 8. Quick Reference: When to Use What

| Use Case | Best Choice |
|---|---|
| Product catalog (read-heavy) | PostgreSQL + Redis cache-aside |
| Order records (ACID) | PostgreSQL |
| Driver location tracking | Cassandra |
| Real-time leaderboard | Redis Sorted Set |
| Full-text product search | Elasticsearch |
| Image/video storage | S3 + CloudFront |
| Session data | Redis |
| Activity feed | Cassandra |
| Financial transactions | PostgreSQL with strict ACID |
| IoT telemetry | Cassandra / TimescaleDB |
| Graph relationships | Neo4j |
| Event streaming | Kafka |
| Task queues | RabbitMQ / SQS |
| CDC for cache invalidation | Debezium → Kafka |

---

## 9. The Universal Architect Answer Pattern

For any "should I use X or Y?" question:

```
1. Clarify the requirement
   "What's the read/write ratio?"
   "What's the scale we're targeting?"
   "What are the consistency requirements?"

2. Identify the constraints
   "Strong consistency needed?"
   "Geographic distribution?"
   "Team expertise?"

3. State the trade-off explicitly
   "X gives us A and B, but sacrifices C."
   "Y gives us C, but sacrifices A and B."

4. Make the recommendation
   "Given your constraints, I'd choose X because..."

5. Plan the escape hatch
   "If we hit limit Z, we'd need to migrate to..."
```

---

## 10. Trade-off Mental Model

Every architectural choice trades off across these axes:

| Axis | Pick One Side |
|---|---|
| Consistency ↔ Availability | CAP theorem |
| Latency ↔ Throughput | Optimize for one |
| Read ↔ Write performance | LSM vs B-Tree |
| Cost ↔ Performance | Tier appropriately |
| Simplicity ↔ Scalability | Start simple |
| Flexibility ↔ Performance | Schema design |

**Always articulate the trade-off in your answer.**
