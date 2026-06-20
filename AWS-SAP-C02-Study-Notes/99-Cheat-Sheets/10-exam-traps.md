# Common Exam Traps — Cheat Sheet

> 🔥 **The single most valuable doc for the last week.** These are the patterns where wrong answers are most plausible.

---

## 🎯 The Anti-Pattern Catalog

These are scenarios where the "obvious" answer is WRONG. Test designers love these.

---

## TRAP 1: Aurora Global ≠ Active-Active Writes

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Need writes from multiple regions with low latency..."

WRONG ANSWER: Aurora Global Database
RIGHT ANSWER: DynamoDB Global Tables

WHY:
─────────────────────────────────────────────
Aurora Global = ACTIVE-PASSIVE (writes go to primary region)
DynamoDB Global Tables = ACTIVE-ACTIVE (writes any region)

EXCEPTION:
─────────────────────────────────────────────
Aurora Global works if "low write latency from one region
+ reads from many" — that's still active-passive but solves
read latency globally.
```

---

## TRAP 2: VPC Peering Is Not Transitive

```
QUESTION PHRASING:
─────────────────────────────────────────────
"VPC A peers to B, B peers to C. A wants to reach C..."

WRONG ANSWER: It just works (transitive)
RIGHT ANSWER: It does NOT work. You need:
              - Direct peering A↔C, OR
              - Transit Gateway (which IS transitive)

WHY:
─────────────────────────────────────────────
VPC peering is point-to-point only.
For 6+ VPCs, the math gets ugly (N²/2 peerings).
TGW is the answer for scale.
```

---

## TRAP 3: EC2 Default Metrics Don't Include Memory

```
QUESTION PHRASING:
─────────────────────────────────────────────
"App is slow. CloudWatch shows CPU at 30% but app
is unresponsive. How to monitor RAM?"

WRONG ANSWER: Default CloudWatch metrics already track it
RIGHT ANSWER: Install CloudWatch Agent

WHY:
─────────────────────────────────────────────
EC2 default metrics: CPU, network, disk I/O — that's it.
Memory and disk SPACE require CloudWatch Agent.
Same for swap, processes, custom app metrics.
```

---

## TRAP 4: Direct Connect Is NOT Encrypted by Default

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Need a private, encrypted connection from on-prem to AWS
with 10 Gbps consistent bandwidth..."

WRONG ANSWER: Direct Connect alone
RIGHT ANSWER: Direct Connect + IPsec VPN (VPN over DX)
              OR: Direct Connect with MACsec (10G/100G dedicated)
              OR: Application-level TLS

WHY:
─────────────────────────────────────────────
DX is a PRIVATE physical line but unencrypted.
Adding IPsec gives you bandwidth + encryption.
```

---

## TRAP 5: RDS Encryption Must Be Set At Creation

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Need to encrypt an existing unencrypted RDS database..."

WRONG ANSWER: Modify the instance to enable encryption
RIGHT ANSWER: 
  1. Snapshot the unencrypted instance
  2. Copy the snapshot WITH encryption enabled (KMS)
  3. Restore from the encrypted snapshot (creates new instance)
  4. Switch app to new instance, delete old one

WHY:
─────────────────────────────────────────────
RDS encryption can ONLY be enabled at creation time.
There's no "encrypt in place" option.
```

---

## TRAP 6: Security Group vs NACL Stateful Confusion

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Configured allow inbound on port 443. Outbound is restricted.
Why aren't responses going out?"

ANALYSIS:
─────────────────────────────────────────────
Security Group is STATEFUL:
  Allowing inbound 443 → response goes out automatically
  
NACL is STATELESS:
  Allowing inbound 443 → ALSO need outbound rule for ephemeral ports
  (typically 1024-65535) for the response

EXAM TRAP:
─────────────────────────────────────────────
If NACLs are misconfigured, traffic breaks even with
correct Security Groups. Always check BOTH layers.
```

---

## TRAP 7: NAT Gateway Costs

```
QUESTION PHRASING:
─────────────────────────────────────────────
"VPC has high data transfer costs through NAT. How to reduce?"

WRONG ANSWER: Reduce internet egress
RIGHT ANSWER: Use VPC Endpoints (Gateway for S3/DDB are FREE!)

WHY:
─────────────────────────────────────────────
NAT Gateway charges $0.045/GB for data processing.
Traffic to S3/DynamoDB through NAT = wasted money.
Gateway VPC Endpoints route to S3/DDB without NAT.
Interface Endpoints for other services (cheaper than NAT).
```

---

## TRAP 8: Lambda VPC Cold Starts (No Longer A Trap)

```
OUTDATED ADVICE: "Don't put Lambda in VPC — slow cold starts"
CURRENT REALITY: Hyperplane ENI makes VPC Lambda < 1s

QUESTION TRAP:
─────────────────────────────────────────────
"Old documentation says to avoid VPC for Lambda..."

CURRENT ANSWER:
─────────────────────────────────────────────
Modern Lambda in VPC is fine.
But: VPC subnets still need NAT or VPC Endpoints
for internet/AWS API access.
```

---

## TRAP 9: S3 One Zone-IA Risk

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Want cheapest S3 storage class..."

WRONG ANSWER: One Zone-IA for everything
RIGHT ANSWER: Depends on re-creatability

WHY:
─────────────────────────────────────────────
One Zone-IA stores in SINGLE AZ.
If that AZ has a disaster → DATA LOST.

USE ONE ZONE-IA ONLY FOR:
- Re-creatable data (logs, derived data)
- Secondary copies
- NOT for "only copy" of important data
```

---

## TRAP 10: KMS Cross-Account Both Sides Required

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Account B can't decrypt objects in Account A's S3 bucket
encrypted with A's CMK..."

CHECK ALL FOUR:
─────────────────────────────────────────────
1. ✅ Bucket policy in A allows B's principal s3:GetObject?
2. ✅ KMS key policy in A allows B's principal kms:Decrypt?
3. ✅ IAM policy in B allows kms:Decrypt on A's CMK ARN?
4. ✅ IAM policy in B allows s3:GetObject on A's bucket?

ALL FOUR REQUIRED. Failing any one = denied.
```

---

## TRAP 11: SQS FIFO Throughput Limits

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Need 50,000 messages/sec with strict ordering..."

WRONG ANSWER: SQS FIFO (assumes unlimited like Standard)
RIGHT ANSWER: 
  Default FIFO: 300 TPS (3,000 with batching)
  High throughput mode: 70,000+ TPS (US/EU/Asia regions)
  
  OR: Use Kinesis with multiple shards if Kafka-like
      semantics fit
```

---

## TRAP 12: Kinesis Ordering Per Shard, Not Global

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Need strict ordering for all events in a Kinesis stream..."

WRONG ANSWER: Kinesis preserves order globally
RIGHT ANSWER: Order is preserved WITHIN a shard

WHY:
─────────────────────────────────────────────
Records with the same partition key go to the same shard
→ ordering preserved within that key

For global order: 1 shard only (limits throughput to 1 MB/s in)
For partitioned order: design partition keys carefully

EXAM PATTERN:
─────────────────────────────────────────────
If question says "events from each user must be ordered" →
partition key = UserId works fine.
If question says "all events must be ordered globally" →
trap, that's a single-shard / SQS FIFO problem.
```

---

## TRAP 13: SCP Doesn't Grant — Only Denies

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Apply SCP to grant read-only S3 to an OU..."

WRONG ANSWER: SCP can grant permissions
RIGHT ANSWER: SCPs only RESTRICT what IAM can grant

WHY:
─────────────────────────────────────────────
SCP is a guardrail / max-perm boundary.
You still need IAM policies in each account to actually grant.
SCP says: "These actions are FORBIDDEN regardless of IAM."
```

---

## TRAP 14: Read Replica vs Multi-AZ

```
COMMON CONFUSION:
─────────────────────────────────────────────
RDS Multi-AZ:
  - Standby in different AZ
  - SYNCHRONOUS replication
  - For HIGH AVAILABILITY (failover)
  - Cannot be read from (standby is invisible)
  
RDS Read Replica:
  - Replica in same/different AZ/region
  - ASYNCHRONOUS replication
  - For READ SCALING
  - Can be promoted to primary manually

QUESTION PATTERNS:
─────────────────────────────────────────────
"Need HA with auto-failover" → Multi-AZ
"Need to scale reads" → Read Replicas
"Need both" → Multi-AZ + Read Replicas (yes, you can)
```

---

## TRAP 15: DynamoDB Hot Partitions

```
QUESTION PHRASING:
─────────────────────────────────────────────
"DynamoDB table is throttling despite low total RCU usage..."

CAUSE: Hot partition (one PK getting all traffic)

WRONG FIX: Increase provisioned capacity
RIGHT FIX: 
  1. Better partition key design (higher cardinality)
  2. OR write-sharding (PK = "userId#randomBucket")
  3. OR use DAX to absorb hot reads
  
EXAM SIGNAL:
─────────────────────────────────────────────
PK like "2026-Month" or "GameDate" = HOT PARTITION TRAP
```

---

## TRAP 16: CloudFront Default Behavior

```
QUESTION PHRASING:
─────────────────────────────────────────────
"CloudFront serving API responses but they appear stale..."

CAUSE: Default caching behavior caches GETs

FIX:
─────────────────────────────────────────────
- Set Cache-Control headers from origin
- OR use cache behavior with TTL=0
- OR disable caching for dynamic API path
- OR use Lambda@Edge / CloudFront Functions for custom logic

EXAM SIGNAL:
─────────────────────────────────────────────
"Dynamic content" + "stale data" → CloudFront caching issue
```

---

## TRAP 17: Route 53 Failover Requires Health Checks

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Set up Route 53 failover but it didn't fail over..."

CAUSE: Failover routing requires HEALTH CHECKS
       Without health checks, R53 doesn't know primary is down

FIX:
─────────────────────────────────────────────
Configure Route 53 Health Checks:
- HTTP/HTTPS endpoint check
- Or calculated health (combine multiple)
- Or CloudWatch alarm-based
Then associate with the primary record.
```

---

## TRAP 18: ALB vs NLB vs CLB

```
TRAP: Picking the wrong load balancer

ALB (Application LB):
─────────────────────────────────────────────
✅ HTTP/HTTPS (Layer 7)
✅ Path-based, host-based routing
✅ WebSockets, gRPC
✅ Authentication integration (Cognito, OIDC)
❌ Static IP (use NLB for that)
❌ Non-HTTP

NLB (Network LB):
─────────────────────────────────────────────
✅ TCP/UDP/TLS (Layer 4)
✅ Static IPs (or EIPs)
✅ Extreme performance
✅ Lower latency
✅ Preserves source IP
❌ HTTP path routing (use ALB)

GWLB (Gateway LB):
─────────────────────────────────────────────
✅ Insert 3rd-party security appliances
✅ Layer 3 (GENEVE encapsulation)
✅ Use: Inline firewalls, IDS/IPS

CLB (Classic LB - LEGACY):
─────────────────────────────────────────────
❌ Don't use for new deployments
Migration recommended to ALB/NLB
```

---

## TRAP 19: S3 Object Lock Requires Versioning

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Need WORM (Write Once Read Many) for compliance..."

ANSWER COMPONENTS:
─────────────────────────────────────────────
✅ S3 Object Lock — REQUIRES Versioning ENABLED
✅ Modes:
   - Governance: Special permission can delete
   - Compliance: NO ONE can delete (even root!)
✅ Retention: Period or Legal Hold

DON'T FORGET versioning prerequisite!
```

---

## TRAP 20: Snowball Edge Costs Aren't Just Device Rental

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Picking cheapest transfer for 100 TB..."

CONSIDER ALL COSTS:
─────────────────────────────────────────────
Snowball costs:
- Device rental (per device)
- Shipping (round-trip)
- Service fee
- Data egress for return

Internet transfer costs:
- Bandwidth time
- Possible upgrade to faster link

WINNER ISN'T ALWAYS OBVIOUS:
─────────────────────────────────────────────
Small data + fast link = internet wins
Large data + slow link = Snowball wins
Network unavailable = Snowball is the only option
```

---

## TRAP 21: Auto Scaling Group Without Min Healthy

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Auto Scaling Group has 5 instances. During refresh, 
all are recycling simultaneously and the app has outages..."

FIX:
─────────────────────────────────────────────
Set UpdatePolicy with:
- MinInstancesInService: 4 (keep 4 healthy at all times)
- MaxBatchSize: 1 (rotate one at a time)
- PauseTime: PT5M (wait between rotations)

Or use Instance Refresh with healthy percentage settings.
```

---

## TRAP 22: EFS Mount Performance — Throughput Mode

```
QUESTION PHRASING:
─────────────────────────────────────────────
"EFS is slow despite many files..."

CAUSE: Burst credit exhaustion (Bursting Throughput mode)

FIX:
─────────────────────────────────────────────
Switch to ELASTIC throughput mode (modern default):
- Auto-scales up to 10 GB/s read, 3 GB/s write
- Pay per request
- No credit system

OR: Provisioned mode (pay extra for guaranteed)
```

---

## TRAP 23: Glacier Retrieval Tier ≠ Storage Class

```
CONFUSION:
─────────────────────────────────────────────
Storage Classes:
  - Glacier INSTANT Retrieval (milliseconds)
  - Glacier FLEXIBLE Retrieval (mins-hours, retrieve tier choice)
  - Glacier DEEP ARCHIVE (12-48 hours)

Retrieval Tiers (only Flexible has these):
  - Expedited (1-5 min)
  - Standard (3-5 hours)
  - Bulk (5-12 hours)

EXAM TRAP:
─────────────────────────────────────────────
"Need 5-min retrieval" — could be:
✅ Glacier Instant (always ms - cheaper storage if quarterly access)
✅ Glacier Flexible + Expedited (rare, but possible)
```

---

## TRAP 24: Step Functions Standard vs Express

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Need orchestration of 100,000 short Lambda invocations/sec..."

STANDARD:
─────────────────────────────────────────────
- Up to 1 year per execution
- Exactly-once
- Audit-friendly
- $25 per million state transitions

EXPRESS:
─────────────────────────────────────────────
- Up to 5 minutes per execution
- At-least-once
- High-throughput (100k/sec+)
- $1 per million invocations
- Use: High-volume event processing
```

---

## TRAP 25: API Gateway REST vs HTTP API

```
REST API (legacy / feature-rich):
─────────────────────────────────────────────
- More features (request validation, transformation)
- Edge-optimized, Regional, or Private
- API Keys, usage plans
- More expensive

HTTP API (modern, cheaper):
─────────────────────────────────────────────
- 70% cheaper than REST
- Lower latency
- Less features (no transformation, etc.)
- Use for simple Lambda backends

WEBSOCKET API:
─────────────────────────────────────────────
- Real-time bidirectional
- Use: Chat, gaming, IoT

EXAM PATTERN:
─────────────────────────────────────────────
"Simple Lambda backend, lowest cost" → HTTP API
"Need request transformation" → REST API
"Real-time updates to clients" → WebSocket API
```

---

## TRAP 26: AWS Backup vs Service-Specific Backup

```
QUESTION PHRASING:
─────────────────────────────────────────────
"Centralize backups across RDS, EBS, EFS, DynamoDB..."

WRONG: Configure each service's backup separately
RIGHT: AWS Backup (single dashboard, policies, retention)

USE AWS BACKUP FOR:
─────────────────────────────────────────────
✅ Cross-service backup management
✅ Cross-account backup sharing
✅ Cross-region backup copies
✅ Backup vault locks (immutable backups)
✅ Compliance reporting
```

---

## TRAP 27: Lambda Reserved vs Provisioned Concurrency

```
RESERVED:
─────────────────────────────────────────────
- MAXIMUM concurrent executions for this function
- Protects DOWNSTREAM (e.g., DB connections)
- Function can be throttled if exceeded
- FREE (just allocation)

PROVISIONED:
─────────────────────────────────────────────
- PRE-WARMED execution environments
- Eliminates cold starts
- PAY for them whether used or not

DON'T CONFUSE:
─────────────────────────────────────────────
"Limit concurrency"     → Reserved
"Eliminate cold starts" → Provisioned
"Both"                  → Reserved + Provisioned together
```

---

## ⚡ The Trap Recall Card

```
┌─────────────────────────────────────────────────┐
│ TOP TRAPS                                        │
├─────────────────────────────────────────────────┤
│ 1. Aurora Global = Active-Passive (not AA)       │
│ 2. VPC Peering = NOT transitive                  │
│ 3. EC2 default metrics = NO memory               │
│ 4. Direct Connect = NOT encrypted                │
│ 5. RDS encryption at creation only               │
│ 6. NACLs stateless (need ephemeral rule)         │
│ 7. NAT $$ — use VPC Endpoints                    │
│ 8. KMS cross-account = 4 policies                │
│ 9. SQS FIFO 300 TPS default                      │
│ 10. SCPs deny only, never grant                  │
│ 11. Multi-AZ ≠ Read Replica                      │
│ 12. DDB Hot Partition = PK design issue          │
│ 13. CloudFront caches by default                 │
│ 14. R53 Failover needs health checks             │
│ 15. ALB vs NLB by use case                       │
│ 16. Object Lock needs Versioning                 │
│ 17. ASG refresh needs UpdatePolicy               │
│ 18. EFS Elastic throughput is the modern default │
│ 19. Step Func Express for high throughput        │
│ 20. HTTP API cheaper than REST API               │
└─────────────────────────────────────────────────┘
```

---

## 🎯 The Meta-Strategy

```
WHEN YOU SEE THE EXAM:
─────────────────────────────────────────────

1. Read carefully for QUALIFIERS:
   "minimum cost" / "maximum availability" / "lowest RTO"
   These eliminate many "technically works" answers.

2. Look for UNNECESSARY COMPLEXITY:
   If two options solve the problem and one is simpler,
   the simpler one is usually correct.

3. Watch for INCORRECT PREMISES:
   "Use VPC peering across 30 VPCs" — wrong, even if 
   theoretically doable. TGW is the right tool.

4. NEVER pick options that require:
   - Sharing IAM credentials (always use roles)
   - Manual operations when automation exists
   - Public S3 buckets for sensitive data
   - Disabling security features for convenience

5. WHEN STUCK: Pick the most AWS-native solution.
   E.g. between Apache Kafka self-managed and MSK,
   pick MSK (the managed service).
```
