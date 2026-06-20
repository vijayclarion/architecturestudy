# Cost Optimization Strategies — Cheat Sheet

> 🔥 **Cost optimization appears in Domain 1, 3, and 4 — at least 20% of total exam.**

---

## 🎯 The 4 Cost Levers

```
1. COMPUTE PRICING (biggest single $$ pool)
2. STORAGE TIERING (data has a lifecycle)
3. DATA TRANSFER (often forgotten — adds up)
4. WORKLOAD RIGHTSIZING (don't pay for unused)
```

---

## 💰 1. COMPUTE — Pricing Model Ladder

```
   CHEAPEST                                          MOST FLEXIBLE
       ↑                                                  ↑
 ┌─────┴─────┐  ┌─────────────┐  ┌────────────┐  ┌─────┴─────┐
 │   SPOT    │  │ SAVINGS PLAN│  │ RESERVED   │  │ ON-DEMAND │
 │ Up to 90% │  │ Up to 72%   │  │ Up to 72%  │  │  0% off   │
 │   off     │  │   off       │  │   off      │  │           │
 └───────────┘  └─────────────┘  └────────────┘  └───────────┘
   Interrupts    1-3 yr commit    1-3 yr commit    No commit
   any time      (flexible)       (more rigid)    Pay-as-you-go
```

### Compute Savings Plan — The Modern Default

```
COMPUTE SAVINGS PLAN — flexibility champion:
─────────────────────────────────────────────
✅ Covers EC2 in any region
✅ Covers any instance family/size
✅ Covers Linux/Windows
✅ Covers Fargate
✅ Covers Lambda
✅ Up to 66% off
✅ 1 or 3 year commit, with All/Partial/No upfront

vs Reserved Instances:
─────────────────────────────────────────────
RIs locked to family + region + OS
Convertible RIs can change (less discount)
Standard RIs can be sold on Reserved Instance Marketplace

VERDICT: Use Compute SP for baseline, RI only if you
        also need capacity reservation guarantee.
```

### Spot Best Practices

```
SPOT INSTANCES:
─────────────────────────────────────────────
✅ Use Spot Fleet with DIVERSIFIED instance types
✅ Use "capacity-optimized" allocation strategy
   (least likely to be interrupted)
✅ Listen for 2-minute interruption notice
✅ Pair with Auto Scaling Group + Mixed Instances Policy
✅ Use multiple AZs

USE CASES (interruption tolerant):
─────────────────────────────────────────────
✅ Batch jobs
✅ CI/CD builds
✅ Data processing (Spark, EMR)
✅ Stateless web (with proper ASG fleet diversity)
✅ ML training (with checkpoints)

DON'T:
─────────────────────────────────────────────
❌ Production databases on Spot
❌ Stateful single-instance workloads
❌ Long-running tasks > 1 hour without checkpoints
```

### Right-Sizing

```
TOOLS:
─────────────────────────────────────────────
- AWS Compute Optimizer
  → ML analysis of CloudWatch metrics
  → Recommends smaller instances if underutilized
  → Covers EC2, EBS, Lambda, ASG

- Trusted Advisor (cost checks)
  → Idle instances (low CPU)
  → Underutilized EBS volumes
  → Unattached EIPs

- Cost Explorer Rightsizing Recommendations
  → Builds on Compute Optimizer

DEFAULT BAD PATTERN:
─────────────────────────────────────────────
Engineer picks m5.large "just in case"
Avg CPU: 5%
Should be t3.small (10x cheaper)
```

---

## 💾 2. STORAGE — Tier and Lifecycle

### S3 Lifecycle Pattern

```yaml
Lifecycle:
  TransitionToStandardIA:    Day 30   ← Standard → Standard-IA
  TransitionToGlacierIR:      Day 90   ← Standard-IA → Glacier Instant
  TransitionToGlacierFlex:    Day 365  ← → Glacier Flexible
  TransitionToDeepArchive:    Day 730  ← → Glacier Deep Archive
  Expire:                     Day 2555 ← Delete after 7 years
```

### Storage Cost Ranking (per GB/month)

```
$0.023  Standard
$0.0125 Standard-IA
$0.01   One Zone-IA
$0.004  Glacier Instant Retrieval
$0.0036 Glacier Flexible Retrieval
$0.00099 Glacier Deep Archive    ← 23x cheaper than Standard!
```

### EBS Cost Tips

```
✅ Use gp3 over gp2 — 20% cheaper
✅ Right-size volumes (delete unused, snapshot then delete)
✅ Snapshots cost less than running volumes
✅ Cold workloads → sc1 (cheapest)
✅ Delete unattached EBS volumes
✅ Schedule snapshot deletion (DLM)
```

### Intelligent-Tiering — Set It and Forget It

```
S3 INTELLIGENT-TIERING:
─────────────────────────────────────────────
✅ Auto-moves objects between tiers based on access
✅ No retrieval fees between Frequent/Infrequent
✅ Small monitoring fee per object
✅ Modern default for "unknown access" data

WHEN: You don't want to analyze access patterns
WHEN NOT: You know all data is hot or all cold
```

---

## 🌐 3. DATA TRANSFER — The Hidden Cost

### The Cost Hierarchy

```
FREE (or cheapest):
─────────────────────────────────────────────
✅ Same AZ, same VPC — free
✅ Internet → EC2 inbound — free
✅ S3 → CloudFront — free
✅ Within S3 same region — free
✅ Service endpoints in same region — free

CHEAPER:
─────────────────────────────────────────────
$$ Cross-AZ within region (~$0.01/GB each direction)
$$ Inter-region replication (~$0.02/GB)

EXPENSIVE:
─────────────────────────────────────────────
$$$ EC2 → Internet outbound (~$0.09/GB)
$$$ CloudFront origin fetches (when not cached)
$$$ NAT Gateway data processing ($0.045/GB!) ⚠️
```

### NAT Gateway — Sneaky Cost

```
NAT GATEWAY CHARGES:
─────────────────────────────────────────────
$0.045 per hour (always running)
+
$0.045 per GB DATA PROCESSED through it

Example: 10 TB through NAT = $450 in data processing alone!

FIX:
─────────────────────────────────────────────
✅ Use VPC Endpoints for AWS services (free for S3/DDB, 
   cheaper for others)
✅ Cache data closer to apps to reduce egress
✅ Consider co-locating producers and consumers
```

### CloudFront for Cost Reduction

```
CLOUDFRONT REDUCES:
─────────────────────────────────────────────
✅ Origin egress (cached responses don't fetch)
✅ Internet-out costs (CloudFront has cheaper data rates)
✅ Geographic latency (better UX, fewer abandonment)

PATTERN:
─────────────────────────────────────────────
S3 → CloudFront → Users
Origin Shield to further reduce origin fetches
```

---

## 📊 4. WORKLOAD RIGHTSIZING

### Auto Scaling Everywhere

```
ANYWHERE YOU CAN AUTO SCALE — DO:
─────────────────────────────────────────────
✅ EC2 ASGs (target tracking)
✅ ECS Service Auto Scaling
✅ Lambda Provisioned Concurrency Auto Scaling
✅ DynamoDB Auto Scaling
✅ Aurora Read Replica Auto Scaling
✅ ElastiCache cluster auto scaling
✅ EKS Karpenter / Cluster Autoscaler

GOAL: Pay only for what you actually need RIGHT NOW.
```

### Serverless = Cost Right-Sizing Automatic

```
SERVERLESS = PAY PER USE:
─────────────────────────────────────────────
- Lambda: per ms execution
- Fargate: per task per second
- Aurora Serverless v2: per ACU-second
- DynamoDB On-Demand: per request
- Athena: per TB scanned

ELIMINATES IDLE COST entirely.
Best for variable workloads.
```

---

## 🎯 5. ORGANIZATIONAL TOOLS

### Cost Visibility

```
COST EXPLORER:
─────────────────────────────────────────────
✅ Visualize costs by service, account, tag
✅ Forecasts
✅ Reserved/Savings Plan recommendations
✅ Right-sizing recommendations

AWS BUDGETS:
─────────────────────────────────────────────
✅ Set monthly $$ thresholds
✅ Alert via SNS / email at 50%, 80%, 100%
✅ Budget actions (auto-stop EC2 if over budget!)

COST ALLOCATION TAGS:
─────────────────────────────────────────────
✅ Tag every resource (Project, Owner, Environment)
✅ Enable in billing console
✅ Group costs by tags in Cost Explorer

COST AND USAGE REPORT (CUR):
─────────────────────────────────────────────
✅ Detailed daily/hourly cost data → S3
✅ Query with Athena for custom analysis
✅ Foundation for FinOps practice

CONSOLIDATED BILLING (via Organizations):
─────────────────────────────────────────────
✅ Volume discounts across accounts
✅ RIs shared across accounts
✅ Savings Plans shared across accounts
✅ One bill for the whole org
```

---

## 🎯 Common Exam Patterns

### Pattern 1: Predictable Workload Cost Reduction

> "EC2 fleet runs 24/7 at known capacity. Want to cut costs."

✅ **Compute Savings Plans** for 1-3 years (up to 66% off)
- Don't lock into Reserved Instances unless capacity reservation matters
- Compute SP is flexible across regions and instance families

---

### Pattern 2: Spiky Workload Cost

> "Batch processing runs nights and weekends. Tolerates interruption."

✅ **Spot Instances with Spot Fleet** + diversified instance types
- 70-90% savings
- ASG auto-replaces interrupted instances

---

### Pattern 3: Massive S3 Storage Bill

> "PB of data in S3 Standard. Most is rarely accessed."

✅ **S3 Lifecycle Policy** to transition to IA → Glacier
✅ Use **S3 Storage Lens** to identify cold data
✅ Use **S3 Intelligent-Tiering** for unknown patterns
- Can reduce storage cost by 80-95% for cold data

---

### Pattern 4: NAT Gateway Surprise

> "VPC has high NAT Gateway data processing costs."

✅ Use **Gateway VPC Endpoints** for S3 and DynamoDB (FREE)
✅ Use **Interface Endpoints** for other AWS services (cheaper than NAT data)
✅ Audit what's actually traversing NAT — may not need internet for AWS services

---

### Pattern 5: Idle/Forgotten Resources

> "Cost is growing without obvious reason."

✅ **AWS Trusted Advisor** — flags idle/underutilized resources
✅ **Compute Optimizer** — rightsizing recommendations
✅ **Cost Explorer** — drill into cost growth
✅ **AWS Budgets** — actions to auto-stop over-budget resources

---

### Pattern 6: Cross-Account Cost Attribution

> "10 teams, need to show each their AWS spend."

✅ **AWS Organizations + Consolidated Billing**
✅ **Cost Allocation Tags** (Project, Team, CostCenter)
✅ **Cost Categories** for advanced grouping
✅ **CUR + Athena** for custom reports

---

### Pattern 7: Multi-Region Active-Active is Too Expensive

> "2x infrastructure for DR is costing too much. RTO of 30 min is OK."

✅ **Downgrade from Multi-Site Active-Active to Pilot Light or Warm Standby**
- Pilot Light: DB always running, app off → minutes RTO, low cost
- Warm Standby: scaled-down version running → faster failover

---

## ⚡ Cost Optimization Quick Recall Card

```
┌─────────────────────────────────────────────────┐
│ COMPUTE                                          │
├─────────────────────────────────────────────────┤
│ Baseline 24/7        → Compute Savings Plan      │
│ Spiky workload       → On-Demand + ASG           │
│ Interruption OK      → Spot Fleet                │
│ BYOL / compliance    → Dedicated Host            │
│ Capacity guarantee   → Capacity Reservation      │
│ Right-size           → Compute Optimizer         │
├─────────────────────────────────────────────────┤
│ STORAGE                                          │
├─────────────────────────────────────────────────┤
│ Cold data            → IA / Glacier              │
│ Unknown pattern      → Intelligent-Tiering       │
│ Re-creatable         → One Zone-IA               │
│ Long archive         → Glacier Deep Archive      │
│ Lifecycle policies   → Automate transitions      │
├─────────────────────────────────────────────────┤
│ DATA TRANSFER                                    │
├─────────────────────────────────────────────────┤
│ Avoid NAT for AWS    → VPC Endpoints             │
│ Reduce internet egress→ CloudFront caching       │
│ Cross-AZ minimize    → Co-locate services        │
├─────────────────────────────────────────────────┤
│ VISIBILITY                                       │
├─────────────────────────────────────────────────┤
│ See costs            → Cost Explorer             │
│ Detailed breakdown   → Cost & Usage Report (S3)  │
│ Set limits           → AWS Budgets + Actions     │
│ Tag costs            → Cost Allocation Tags      │
│ Org-wide             → Consolidated Billing      │
└─────────────────────────────────────────────────┘
```
