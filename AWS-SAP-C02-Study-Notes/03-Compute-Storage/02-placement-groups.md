# EC2 Placement Groups

> 🔥 **Niche but tested topic.** Know the three types and their use cases.

## Problem It Solves
By default, EC2 places instances wherever it has capacity — across AZs, racks, hardware. For most workloads that's fine. But some workloads need explicit control:
- HPC clusters need instances physically close together (low latency)
- Critical workloads need instances spread across racks (avoid hardware failure)
- Big distributed databases want partition-aware placement

**Placement Groups give you that control over how EC2 lays your instances onto the physical infrastructure (racks, hardware, AZs).**

---

## ⚠️ Myth Buster (read this first)

```
THERE ARE ONLY THREE PLACEMENT GROUP STRATEGIES:
─────────────────────────────────────────────
   Cluster  •  Partition  •  Spread

❌ There is NO "system" placement group.
❌ "Host vs Rack" is NOT a 4th type — it's the SPREAD LEVEL
   (a sub-setting of the Spread strategy). See Spread section.
✅ Bare-metal (.metal) instances CAN go in placement groups
   (e.g. cluster) — but that's just an instance type choice,
   not a separate placement group type.
```

If someone in your audience says "system placement group," gently redirect: they're likely thinking of the spread level or mixing it up with something else.

---

## The Three Types

```mermaid
flowchart TD
    PG[Placement Groups]
    PG --> CLUS[Cluster<br/>Close together<br/>= lowest latency]
    PG --> PART[Partition<br/>Logical separation<br/>= fault isolation at scale]
    PG --> SPREAD[Spread<br/>Maximum separation<br/>= each instance independent]
```

| | **Cluster** | **Partition** | **Spread** |
|---|---|---|---|
| Goal | Lowest latency | Distributed system fault isolation | Maximum hardware independence |
| Placement | Same AZ, close hardware/rack | Multiple partitions, distinct racks | Each instance on different hardware |
| Network | 10/25 Gbps single-flow | Standard | Standard |
| Use case | HPC, ML training | Hadoop, Cassandra, HDFS | Critical small fleets |
| Max instances | Limited (depends on instance type) | Many (7 partitions × N per AZ) | 7 per AZ |
| Can span AZs? | ❌ Single AZ only | ✅ Yes | ✅ Yes |

---

## 🧠 Concepts Clarified (the intuition behind it)

This section captures the "why," not just the "what" — the stuff that's easy to mix up.

### 1. What *is* a partition, physically?

A **partition is a bunch of EC2 instances that DO share underlying hardware/racks with each other.** Different partitions do NOT share hardware — they sit on separate racks, possibly in different AZs.

```mermaid
flowchart TB
    subgraph PG[One Partition Placement Group]
        subgraph P1[Partition 1 — Rack X]
            A1[Instance A1]
            A2[Instance A2]
        end
        subgraph P2[Partition 2 — Rack Y, maybe diff AZ]
            B1[Instance B1]
            B2[Instance B2]
        end
    end
    A1 -.same rack, close.- A2
    B1 -.same rack, close.- B2
    P1 -. isolated from .- P2
```

- **Within** a partition → instances are together (can fail together).
- **Across** partitions → hardware-isolated (one partition dying ≠ another dying).

### 2. The data-partitioning analogy (good for your talk)

Think of a big distributed database sharded by some key (e.g. `country_id`):

```mermaid
flowchart LR
    subgraph PG[Partition Placement Group]
        direction TB
        subgraph PA[Partition 1]
            C1[Countries 1–10<br/>2 EC2 nodes]
        end
        subgraph PB[Partition 2]
            C2[Countries 11–20<br/>2 EC2 nodes]
        end
        subgraph PC[Partition 3]
            C3[Countries 21–30<br/>2 EC2 nodes]
        end
    end
```

Countries 1–10 live on partition 1's hardware, 11–20 on partition 2's, etc. The app maps shards → partitions so a single rack failure only knocks out one shard's worth of nodes.

### 3. What happens when a partition fails? 🔑

```
A partition's hardware fails
        │
        ▼
Those instances (and the data they hold) become UNAVAILABLE
        │
        ▼
You stay up ONLY IF the app replicated that data to ANOTHER partition
```

**Key takeaway:** the placement group only gives you *physical isolation*. **Redundancy is the app's job** — Cassandra/HDFS/Kafka replicate each shard's data across *different* partitions, so losing partition 1 doesn't lose the data (a replica lives in partition 4, etc.).

> Placement Group = isolation. Replication = survival. You need both.

### 4. Cluster is NOT for high availability ❌

A common trap: "Can I spread a cluster across racks for HA?"

**No.** The whole point of cluster is to pack instances tightly (same rack, same AZ) for lowest latency. Spreading them across racks would throw away the latency benefit — and you'd still be stuck in one AZ. **If you want HA, that's literally what partition and spread are for.**

```
Cluster   → optimize for SPEED   (accepts shared-fate risk)
Partition → optimize for SCALE + ISOLATION
Spread    → optimize for ISOLATION of a few critical nodes
```

### 5. Resilience vs High Availability (these confused me — here's the clean split)

| Term | Plain meaning |
|---|---|
| **High Availability (HA)** | The *service stays up* through a failure, because surviving instances keep serving requests. |
| **Resilience / fault isolation** | A failure *doesn't cascade* — one dead instance can't drag others down, because they share no hardware. |

**Spread gives you both at once:** every instance is on its own hardware, so (a) one failure only takes one instance → resilient, and (b) the rest keep running → available.

```mermaid
flowchart LR
    F[1 hardware fails]
    F --> C{Placement type?}
    C -->|Cluster| CR[Whole group can go down<br/>shared rack 😱]
    C -->|Partition| PR[Only that partition down<br/>others fine ✅]
    C -->|Spread| SR[Only that 1 instance down<br/>everything else fine ✅✅]
```

---

## Cluster Placement Group

```mermaid
flowchart LR
    AZ[Single AZ]
    AZ --> RACK[Single Rack / very close hardware]
    RACK --> N1[Node 1]
    RACK --> N2[Node 2]
    RACK --> N3[Node 3]
    RACK --> NN[Node N]

    N1 <-.10 Gbps low-latency.-> N2
    N1 <-.10 Gbps low-latency.-> N3
    N1 <-.10 Gbps low-latency.-> NN
```

```
CLUSTER PLACEMENT GROUP:
─────────────────────────────────────────────
✅ Instances placed close together
✅ Same Availability Zone (single AZ — cannot span AZs)
✅ Same underlying network/rack
✅ Up to 10 Gbps single-flow throughput (vs 5 Gbps standard)
✅ Low-latency interconnect

LIMITATIONS:
─────────────────────────────────────────────
❌ Single AZ = AZ failure kills entire group
❌ Limited capacity (may fail to launch if capacity tight)
❌ Best to launch all instances at once
❌ NOT for high availability (shared fate by design)

USE CASES:
─────────────────────────────────────────────
✅ HPC clusters (MPI workloads)
✅ Distributed ML training (multi-GPU)
✅ Real-time tightly-coupled apps
✅ Big data with intra-cluster network demands
✅ Financial trading systems

RECOMMENDATION:
─────────────────────────────────────────────
Use ENA (Elastic Network Adapter) instance types
Use same instance type for all (consistent performance)
```

---

## Partition Placement Group

```mermaid
flowchart TB
    PG[Partition PG]
    PG --> P1[Partition 1<br/>Rack X]
    PG --> P2[Partition 2<br/>Rack Y]
    PG --> P3[Partition 3<br/>Rack Z]
    PG --> P4[... up to 7 per AZ]

    P1 --> N1[Node A1]
    P1 --> N2[Node A2]
    P2 --> N3[Node B1]
    P2 --> N4[Node B2]
```

```
PARTITION PLACEMENT GROUP:
─────────────────────────────────────────────
✅ Up to 7 partitions per AZ
✅ Each partition has its own racks (own network + power)
✅ Multiple instances per partition
✅ Different partitions = different physical hardware
✅ Can span multiple AZs (each AZ has its own partitions)

KEY GUARANTEE:
─────────────────────────────────────────────
Failure of one partition's hardware ≠ failure of another's
Distributed system can replicate across partitions for resilience

GOTCHAS:
─────────────────────────────────────────────
• Dedicated Instances → max 2 partitions
• EC2 tries to spread instances evenly across partitions
  but does NOT guarantee perfectly even distribution
• Only worth it if the app is partition-AWARE

USE CASES:
─────────────────────────────────────────────
✅ HDFS (Hadoop Distributed File System)
✅ Cassandra clusters
✅ Kafka brokers
✅ Large distributed databases (50+ nodes)
✅ Big data workloads with partition-aware replication

EXAMPLE:
─────────────────────────────────────────────
Cassandra cluster:
- 7 partitions in us-east-1a
- 7 partitions in us-east-1b
- 7 partitions in us-east-1c
- Replicate data across partitions and AZs
- Single rack failure = at most one partition affected
```

### Partition-Aware Apps

```
WHY PARTITIONS MATTER:
─────────────────────────────────────────────
Distributed systems can query their own partition number:
  http://169.254.169.254/latest/meta-data/placement/partition-number

Use this for replica placement decisions:
- Replica 1 in partition 1
- Replica 2 in partition 4 (different partition)
- Replica 3 in partition 7

Result: Hardware fault in partition 1 doesn't take all replicas.
```

---

## Spread Placement Group

```mermaid
flowchart TB
    PG[Spread PG] --> AZ1[AZ 1]
    PG --> AZ2[AZ 2]
    PG --> AZ3[AZ 3]

    AZ1 --> H1[Hardware A]
    AZ1 --> H2[Hardware B]
    AZ1 --> H3[Hardware C]
    H1 --> I1[Instance 1]
    H2 --> I2[Instance 2]
    H3 --> I3[Instance 3]
```

```
SPREAD PLACEMENT GROUP:
─────────────────────────────────────────────
✅ Each instance on completely separate hardware
✅ Max 7 instances per AZ in the group
✅ Can span multiple AZs (7 per AZ)
✅ MAXIMUM fault isolation
✅ Strictest of the three types
✅ Gives you BOTH high availability AND resilience

GOTCHAS:
─────────────────────────────────────────────
• Hard limit: 7 running instances per AZ per group
  (need more? use multiple spread groups — no cross-group
   spread guarantee, but each group is still spread)
• Not supported for Dedicated Instances
• If not enough unique hardware exists, launch FAILS
  (retry later — EC2 frees up distinct hardware over time)

USE CASES:
─────────────────────────────────────────────
✅ Small fleets where each instance is critical
✅ Master/leader nodes in distributed systems
✅ Quorum-based systems (3-5 critical nodes)
✅ App servers behind an LB where losing one must NOT
   take down the others (no-downtime goal)
✅ Anything where you can't lose multiple at once

EXAMPLE:
─────────────────────────────────────────────
Etcd / Consul / ZooKeeper cluster:
- 5 nodes, each on different hardware
- Single hardware failure = lose 1 node only
- Quorum (3 of 5) still met
```

### Spread Levels  ✅ CORRECTED

> This is the "host vs rack" thing that is NOT a separate placement group type.

```
RACK-LEVEL SPREAD (default / standard):
─────────────────────────────────────────────
Each instance on a distinct RACK (own network + power)
✅ Works in normal AWS Regions
✅ Also works on AWS Outposts
This is what you get in a Region.

HOST-LEVEL SPREAD (Outposts ONLY):
─────────────────────────────────────────────
Each instance on a distinct HOST
❌ Only available on AWS Outposts deployments
```

| Spread level | Where it works | Separates by |
|---|---|---|
| **Rack** (default) | Regions **and** Outposts | Rack |
| **Host** | Outposts **only** | Host |

---

## Comparison Matrix

| Aspect | Cluster | Partition | Spread |
|---|---|---|---|
| Best for | Latency | Distributed DB | Small critical fleet |
| Network throughput | Up to 10 Gbps single flow | Standard | Standard |
| Same AZ requirement | Yes (single AZ) | No (can span AZs) | No (can span AZs) |
| Max instances | Depends on instance/capacity | Many (7 partitions × N) | 7 per AZ |
| Hardware isolation | None (tight together) | Per partition | Per instance |
| Fault tolerance | None (single rack) | Partition-level | Per-instance |
| Latency between instances | Lowest | Higher (partitions) | Higher (spread) |
| Blast radius of 1 HW fault | Whole group | One partition | One instance |

---

## When to Use Each

### Cluster

```
✅ "MPI workload — need lowest latency between nodes"
✅ "Training large ML model across 8 GPUs"
✅ "Financial trading platform — every microsecond counts"
✅ "Big data Spark job, lots of shuffle traffic"
✅ "HPC simulation"
```

### Partition

```
✅ "Hadoop / HDFS cluster with replication"
✅ "Cassandra ring needing hardware isolation"
✅ "Kafka cluster with partition-aware brokers"
✅ "Large distributed database (50+ nodes)"
✅ "Want both isolation AND many instances"
```

### Spread

```
✅ "5-node Consul cluster, can't lose 2 at once"
✅ "Database primary + 2 replicas, all must survive"
✅ "Small but critical service tier"
✅ "Quorum-based system with 3-7 nodes"
```

---

## Mixing & Limitations

```
CAN'T MIX TYPES:
─────────────────────────────────────────────
Each placement group is ONE type
An instance can be in only ONE placement group at a time
Need different types? Use multiple placement groups

INSTANCE TYPE LIMITS:
─────────────────────────────────────────────
Cluster: needs supported instance types (most modern)
Spread: not supported for Dedicated Instances / t2-class
Partition: Dedicated Instances → max 2 partitions

MOVING INSTANCES:
─────────────────────────────────────────────
Can move an existing instance INTO a placement group
(must be STOPPED first), and move it out the same way

CAPACITY ISSUES:
─────────────────────────────────────────────
Cluster: may fail if AZ lacks enough capacity on close hardware
Spread: fails if not enough UNIQUE hardware (retry later)
Mitigation: Launch all at once, use On-Demand Capacity Reservation
```

---

## Combining with Other Tactics

### Cluster + Capacity Reservation

```
ENSURE CLUSTER LAUNCH SUCCEEDS:
─────────────────────────────────────────────
Reserve capacity in advance with On-Demand Capacity Reservation
Specify placement group when reserving
Guarantees the instances can launch when needed
(Note: Capacity Reservations do NOT reserve within a
 partition placement group.)
```

### Partition + ASG

```
AUTO SCALING WITH PARTITIONS:
─────────────────────────────────────────────
ASG can launch into a partition placement group
Distribute new instances across partitions
Maintain partition-aware fault tolerance as you scale
```

---

## When NOT to Use

- ❌ Don't use Cluster for HA workloads (single AZ, shared fate)
- ❌ Don't use Spread for >7 instances per AZ (use Partition)
- ❌ Don't use Partition if you don't have partition-aware app logic
- ❌ Don't use any PG if standard placement is fine (most workloads)

## Trade-offs

| Choice | Gain | Sacrifice |
|---|---|---|
| Cluster | Best network performance | No AZ HA, capacity constraints |
| Partition | Big fleet + hardware isolation | App must be partition-aware |
| Spread | Strongest per-instance isolation | Hard limit of 7/AZ |
| No PG | Maximum flexibility | No control over placement |

## Real World Example

**A trading firm:**
- Latency-critical matching engine: **Cluster PG** in one AZ, with capacity reservation
- 10 Gbps inter-node communication for order matching
- Trade-off: AZ failure = entire matching engine down
- Mitigation: Active-standby matching engines in different AZs (separate cluster PGs)

**A big data company (Cassandra):**
- **Partition PG** with 7 partitions per AZ × 3 AZs
- Cassandra replication strategy aware of partitions
- Configured replicas to span different partitions and AZs
- Single rack failure = 1 partition affected, replication keeps cluster healthy

**A critical microservice:**
- 3-node etcd cluster (cluster state for K8s)
- **Spread PG** ensures each node on different hardware
- Single hardware fault doesn't kill quorum (need 2 of 3)

## 🎯 Exam Soundbite

> "EC2 Placement Groups control instance placement for specific needs. Cluster placement groups put instances physically close for lowest network latency (10 Gbps single-flow) but require a single AZ — use for HPC and tightly-coupled workloads. Partition placement groups divide instances across up to 7 hardware-isolated partitions per AZ — use for large distributed databases like Cassandra, HDFS, and Kafka where the app is partition-aware. Spread placement groups put each instance on completely separate hardware (max 7 per AZ) — use for small critical fleets where you can't tolerate multiple simultaneous failures. There are only THREE strategies — no 'system' type; host vs rack is just the spread level."

---

## ⚡ Quick Recall Card

```
┌─────────────────────────────────────────────────┐
│ PLACEMENT GROUPS  (3 types — no "system" type)   │
├─────────────────────────────────────────────────┤
│ CLUSTER                                          │
│   Same AZ, close hardware → SPEED                │
│   10 Gbps single-flow possible                   │
│   Use: HPC, ML training, low-latency             │
│   ⚠ Not for HA (single AZ, shared fate)          │
│                                                  │
│ PARTITION                                        │
│   Up to 7 partitions per AZ                      │
│   Each partition = isolated hardware/rack        │
│   Use: HDFS, Cassandra, Kafka (big distributed)  │
│   App should be partition-aware + replicate      │
│                                                  │
│ SPREAD                                           │
│   Each instance on different hardware            │
│   Max 7 instances per AZ → HA + resilience       │
│   Use: Small critical fleets (3-7 nodes)         │
│   Levels: rack (default) / host (Outposts only)  │
└─────────────────────────────────────────────────┘
```

## 🎓 Common Exam Patterns

> **"HPC application running MPI, need lowest possible latency between nodes."**
>
> ✅ **Cluster Placement Group** in a single AZ with ENA-enabled instances.

> **"100-node Cassandra cluster needs hardware isolation for fault tolerance."**
>
> ✅ **Partition Placement Group** with 7 partitions × 3 AZs = up to 21 partitions, Cassandra config replicates across partitions.

> **"5-node consul cluster — each node must be on independent hardware."**
>
> ✅ **Spread Placement Group** (max 7 per AZ — 5 fits easily).

> **"Need ~50 highly-available instances per AZ for a Hadoop ETL workload."**
>
> ✅ **Partition Placement Group** (Spread caps at 7/AZ, so it can't fit 50).
