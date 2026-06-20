# Consistent Hashing (Cassandra Foundation)

## Problem It Solves

Traditional hashing (`hash(key) % N`) breaks catastrophically when nodes are added or removed — almost ALL keys remap to different nodes, causing massive data migration. Consistent hashing minimizes data movement when cluster size changes.

## Naive Hashing Problem

```
3 nodes:  key % 3
4 nodes:  key % 4

Adding 1 node → almost every key changes home
→ 75% of data moves ❌
```

## How Consistent Hashing Works

### The Hash Ring

```mermaid
graph TB
    subgraph "Hash Ring 0° to 360°"
        N1[Node A at 90°]
        N2[Node B at 180°]
        N3[Node C at 270°]
        
        K1[Key 'order_123' hashes to 120°]
        K2[Key 'user_456' hashes to 200°]
        K3[Key 'product_789' hashes to 50°]
    end
    
    K1 -.clockwise.-> N2
    K2 -.clockwise.-> N3
    K3 -.clockwise.-> N1
    
    style N1 fill:#e1f5ff
    style N2 fill:#ffe1e1
    style N3 fill:#fff4e1
```

**Rule:** Each key travels **clockwise** to find its node.

### Adding a Node — Minimal Disruption

```mermaid
graph TB
    subgraph "Before adding Node D"
        A1[Node A 90°]
        B1[Node B 180°]
        C1[Node C 270°]
    end
    
    subgraph "After adding Node D at 135°"
        A2[Node A 90°]
        D2[Node D 135°]
        B2[Node B 180°]
        C2[Node C 270°]
    end
    
    Note1[Only data between 90°-135°<br/>moves from Node B to Node D]
    
    style D2 fill:#ccffcc
    style Note1 fill:#ffffcc
```

**Result:** Only ~25% of data moves (proportional to new node's range), not 75%!

## Virtual Nodes (vnodes) — Better Distribution

Real Cassandra uses **virtual nodes** — each physical node owns many small token ranges.

```mermaid
graph TB
    subgraph "Without vnodes - uneven if nodes differ in size"
        N1[Node A: huge range]
        N2[Node B: small range]
        N3[Node C: medium range]
    end
    
    subgraph "With vnodes (256 per node) - even distribution"
        P1[Physical Node 1<br/>256 small ranges scattered]
        P2[Physical Node 2<br/>256 small ranges scattered]
        P3[Physical Node 3<br/>256 small ranges scattered]
    end
    
    style P1 fill:#ccffcc
    style P2 fill:#ccffcc
    style P3 fill:#ccffcc
```

**Benefits of vnodes:**
- Even data distribution even if nodes have different capacity
- Faster rebalancing (parallelized across many small ranges)
- Less hotspotting

## Replication on the Ring

```mermaid
graph LR
    subgraph "Replication Factor = 3"
        K[Key 'order_123' hashes to 120°] --> N2[Node B 180° - PRIMARY]
        N2 -.next clockwise.-> N3[Node C 270° - REPLICA 1]
        N3 -.next clockwise.-> N4[Node D 360° - REPLICA 2]
    end
    
    style N2 fill:#ffcccc
    style N3 fill:#ccffcc
    style N4 fill:#ccffcc
```

Replicas placed on **next N-1 nodes clockwise**. Across racks/datacenters for fault tolerance.

## Gossip Protocol — How Nodes Discover Each Other

```mermaid
sequenceDiagram
    participant A as Node A
    participant B as Node B
    participant C as Node C
    participant D as Node D (new)

    Note over D: Boots up, contacts seed node
    D->>A: Hello, I'm joining
    A->>D: Cluster state
    
    loop Every 1 second
        A->>B: Gossip my state + others I know
        B->>A: My state + others I know
        B->>C: Gossip
        C->>D: Gossip (D now known to C)
    end
    
    Note over A,D: Within seconds, entire cluster knows about Node D
```

### What's in a Gossip Message?

```json
{
  "nodeId": "Node-A",
  "status": "UP",
  "load": "45%",
  "tokens": [90, 450, 780],
  "generation": 12345,
  "schemaVersion": "v3",
  "knownNodes": {
    "Node-B": { "status": "UP", "tokens": [...] },
    "Node-C": { "status": "DOWN", "lastSeen": "..." }
  }
}
```

## Failure Detection via Gossip

```mermaid
stateDiagram-v2
    [*] --> Alive
    Alive --> Suspected: Heartbeat missed
    Suspected --> Alive: Heartbeat resumes
    Suspected --> Down: Timeout exceeded
    Down --> Alive: Node returns + sync
    Down --> Removed: Manual decommission
```

**Phi Accrual Failure Detector** — Cassandra's algorithm:
- Doesn't binary "up/down" — calculates suspicion level (phi)
- Adapts to network conditions
- Higher phi = more likely the node is down

## Full Rebalancing Flow

```mermaid
sequenceDiagram
    participant New as New Node
    participant Seed as Seed Node
    participant Cluster as Existing Cluster
    participant Neighbor as Neighbor Node

    New->>Seed: Contact seed node
    Seed->>New: Current cluster topology
    New->>New: Pick token range (vnodes)
    
    par Gossip broadcast
        New->>Cluster: Announce join (via gossip)
        Cluster->>Cluster: Update routing tables
    end
    
    Neighbor->>New: Stream data for new range
    Note over Neighbor: Only data in new node's range
    
    New->>Cluster: Ready to serve
    
    par Cleanup
        Neighbor->>Neighbor: Delete data no longer owned
    end
```

## Why This Matters for Architects

### Operational Benefits

| Scenario | Traditional Hashing | Consistent Hashing |
|---|---|---|
| Add node | 75%+ data moves | ~1/N data moves |
| Remove node | 75%+ data moves | Only that node's data moves |
| Node failure | All keys remap | Adjacent nodes pick up |
| Heterogeneous nodes | Uneven load | Even via vnodes |

### Real-World Impact

```
100 TB dataset, 10 nodes, add 1 more node:

Traditional hashing:    ~75 TB moves    → days of disruption
Consistent hashing:     ~10 TB moves    → hours
With vnodes:            ~10 TB parallel → minutes
```

## Where Else Consistent Hashing Is Used

- **DynamoDB** (Amazon) — original Dynamo paper
- **Cassandra** — directly inspired by Dynamo
- **Redis Cluster** — slightly different (16384 slots)
- **Memcached client libraries** (ketama hashing)
- **Akamai CDN** — request routing
- **Load balancers** — sticky sessions

## Trade-offs

| Gain | Sacrifice |
|---|---|
| Minimal data movement | More complex than modulo hashing |
| Auto-rebalancing | Eventual consistency during rebalance |
| Linear scaling | Requires coordination protocol (gossip) |
| Heterogeneous nodes (vnodes) | Higher memory for routing tables |

## Common Mistakes

❌ **Confusing consistent hashing with "consistency" in CAP** — unrelated concepts  
❌ **Not using vnodes** — leads to uneven distribution  
❌ **Bad partition key choice** — even consistent hashing can't fix hotspots  
❌ **Ignoring rack/DC awareness** — replicas all in one rack = no fault tolerance  

## Interview Soundbite

> *"Consistent hashing solves the catastrophic rebalancing problem of modulo hashing. Nodes and keys live on a hash ring; keys travel clockwise to their node. Adding a node moves only a fraction of data — proportional to the new node's range. Cassandra extends this with virtual nodes for even distribution across heterogeneous hardware, and gossip protocol for decentralized cluster awareness. The whole system has no master and scales linearly with zero downtime."*
