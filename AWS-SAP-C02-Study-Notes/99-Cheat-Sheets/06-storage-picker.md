# Storage Decision Tree — Cheat Sheet

> 🔥 **One of the most common SAP-C02 questions: "Which storage service?"** Walk this tree to win.

---

## 🌳 The Master Storage Decision Tree

```mermaid
flowchart TD
    START[Need storage?] --> Q1{Access pattern?}
    
    Q1 -->|Block storage<br/>single instance| BLOCK[Block Branch]
    Q1 -->|Shared file storage<br/>many instances| FILE[File Branch]
    Q1 -->|Object storage<br/>API access| OBJ[Object Branch]
    Q1 -->|Archive<br/>rarely accessed| ARCH[Archive Branch]
    Q1 -->|Hybrid<br/>on-prem + cloud| HYBRID[Hybrid Branch]
    Q1 -->|Disaster recovery| DR[DR Branch]
```

---

## 🧱 BLOCK Branch (one instance attaches)

```mermaid
flowchart TD
    BLOCK[Block storage need] --> Q1{Use case?}
    
    Q1 -->|EC2 root/data, default| GP3[gp3 ✅ default]
    Q1 -->|Extreme IOPS database| IO2BE[io2 Block Express]
    Q1 -->|Big data sequential| ST1[st1 - HDD throughput]
    Q1 -->|Cold archive block| SC1[sc1 - HDD cheapest]
    Q1 -->|Cluster shared block<br/>Oracle RAC| IO2MA[io2 Multi-Attach]
    Q1 -->|Temporary scratch| IS[Instance Store NVMe]
```

```
QUICK PICK:
─────────────────────────────────────────────
Default for everything           → gp3
Modern boot volume               → gp3
Mission-critical DB              → io2 Block Express
Big data Hadoop                  → st1
Cold rarely-read block           → sc1
Shared block (Oracle RAC)        → io2 Multi-Attach
Highest IO, temporary            → Instance Store
```

---

## 📁 FILE Branch (multiple instances share)

```mermaid
flowchart TD
    FILE[Shared file storage] --> Q1{OS / protocol?}
    
    Q1 -->|Linux NFS, elastic, multi-AZ| EFS[Amazon EFS]
    Q1 -->|Linux high-perf NFS| FXO[FSx for OpenZFS]
    Q1 -->|Windows SMB + AD| FXW[FSx for Windows File Server]
    Q1 -->|Linux + Windows multi-protocol| FXN[FSx for NetApp ONTAP]
    Q1 -->|HPC / ML parallel filesystem| FXL[FSx for Lustre]
```

```
QUICK PICK:
─────────────────────────────────────────────
General Linux NFS               → EFS
WordPress / CMS multi-instance  → EFS
Windows SMB shares              → FSx for Windows
Mixed Linux + Windows NFS+SMB   → FSx for ONTAP
HPC / Spark / ML training       → FSx for Lustre (S3 linked)
High-perf NFS, cheaper than ONTAP→ FSx for OpenZFS
Container shared volume         → EFS (ECS/EKS supported)
```

### EFS Storage Classes

```
EFS STANDARD            Multi-AZ, hot
EFS STANDARD-IA         Multi-AZ, cold (cheaper)
EFS ONE ZONE            Single AZ, hot
EFS ONE ZONE-IA         Single AZ, cold (cheapest)

Lifecycle management auto-transitions based on access pattern.
```

---

## 📦 OBJECT Branch (S3)

```mermaid
flowchart TD
    OBJ[Object storage] --> Q1{Access frequency?}
    
    Q1 -->|Hot, daily| STD[S3 Standard]
    Q1 -->|Unknown, variable| INT[S3 Intelligent-Tiering ✅]
    Q1 -->|Monthly access| IA[Standard-IA]
    Q1 -->|Monthly, can re-create| OZIA[One Zone-IA]
    Q1 -->|Quarterly, instant| GIR[Glacier Instant Retrieval]
    Q1 -->|Yearly, hours OK| GFR[Glacier Flexible Retrieval]
    Q1 -->|Decades, 12h+ OK| GDA[Glacier Deep Archive]
```

### S3 Storage Class Costs (relative)

```
$$$$$  S3 Standard
$$$$   S3 Standard-IA
$$$    S3 One Zone-IA
$$     S3 Glacier Instant Retrieval
$      S3 Glacier Flexible Retrieval
¢      S3 Glacier Deep Archive  (~23x cheaper than Standard)
```

### S3 Minimum Storage Duration

```
Standard / Int-Tier         None
Standard-IA / One Zone-IA   30 days
Glacier Instant             90 days
Glacier Flexible            90 days
Glacier Deep Archive        180 days

⚠️ Delete before minimum = charged full duration anyway
```

---

## 🗄️ ARCHIVE Branch

```mermaid
flowchart TD
    ARCH[Need archive?] --> Q1{Retrieval time tolerance?}
    
    Q1 -->|Milliseconds<br/>still occasional access| GIR[Glacier Instant Retrieval]
    Q1 -->|Minutes-hours OK| GFR[Glacier Flexible Retrieval]
    Q1 -->|12-48 hours OK<br/>cheapest| GDA[Glacier Deep Archive ✅]
    Q1 -->|Compliance: immutable| GVL[S3 Object Lock + Glacier Vault Lock]
```

```
GLACIER VAULT LOCK:
─────────────────────────────────────────────
WORM (Write Once Read Many) compliance
Locks objects so no one can delete
Required by some financial regulations
SEC Rule 17a-4(f) compliant

S3 OBJECT LOCK (newer, recommended):
─────────────────────────────────────────────
Same WORM concept but native to S3
Governance Mode: special permissions can override
Compliance Mode: NO ONE can delete (even root)
```

---

## 🌉 HYBRID Branch (on-prem + AWS)

```mermaid
flowchart TD
    HYB[Hybrid storage] --> Q1{Pattern?}
    
    Q1 -->|On-prem app needs cloud storage| SG[AWS Storage Gateway]
    Q1 -->|One-time bulk migration| DS[DataSync]
    Q1 -->|Massive offline| SNOW[Snow Family]
    Q1 -->|Continuous DR replication| DRSER[AWS Elastic Disaster Recovery]
    
    SG --> SG1{Gateway type?}
    SG1 -->|S3 via file protocol| FG[S3 File Gateway<br/>NFS/SMB to S3]
    SG1 -->|FSx via SMB protocol| FXG[FSx File Gateway]
    SG1 -->|iSCSI block, cached| VC[Volume Gateway - Cached]
    SG1 -->|iSCSI block, full local| VS[Volume Gateway - Stored]
    SG1 -->|VTL backup replacement| TG[Tape Gateway]
```

### Storage Gateway Types

```
S3 FILE GATEWAY:
─────────────────────────────────────────────
NFS/SMB on-prem → S3 objects in cloud
Files appear as S3 objects (one-to-one)
Local cache for performance
Use: Lift-and-shift file servers to S3

FSx FILE GATEWAY:
─────────────────────────────────────────────
SMB on-prem → FSx for Windows in cloud
For Windows file shares that should live in cloud
Local cache for low-latency reads

VOLUME GATEWAY - CACHED:
─────────────────────────────────────────────
iSCSI block volumes
Primary data in S3, frequently-accessed cached on-prem
Use: Cloud-backed local storage with low latency

VOLUME GATEWAY - STORED:
─────────────────────────────────────────────
iSCSI block volumes
Primary data on-prem, async snapshots to AWS
Use: Local primary with cloud DR

TAPE GATEWAY:
─────────────────────────────────────────────
Replaces physical tape backups
Backup software writes to "virtual tapes"
Tapes auto-archived to Glacier
Use: Eliminate tape library
```

---

## 🚛 BULK / OFFLINE Branch (Snow Family)

```
SNOWCONE        8 TB / 14 TB SSD
                Edge + small data
                Portable

SNOWBALL EDGE   80 TB (storage) or 39 TB + compute
                Standard data migration
                Suitcase size

SNOWBALL EDGE   Includes GPUs and EC2-compatible instances
COMPUTE         Edge ML inference, remote sites

SNOWMOBILE      100 PB
                Truck-based
                Massive data center exit
```

---

## 🎯 Common SAP-C02 Storage Patterns

### Pattern 1: Lift-and-Shift Windows File Server

> "On-prem Windows file server with 5 TB of data and AD authentication. Moving to AWS."

✅ **FSx for Windows File Server** (managed SMB + AD)
- DataSync for initial bulk migration
- Or **FSx File Gateway** if some users stay on-prem

---

### Pattern 2: PB-Scale Data Lake

> "10 PB of varied data — logs, images, JSON. Many access patterns."

✅ **S3 with Intelligent-Tiering** for unknown access patterns
- Lifecycle policies for known archive data
- **Lake Formation** for governance
- **Athena / EMR / Redshift Spectrum** for query

---

### Pattern 3: Database Backups for 7 Years

> "RDS snapshots must be retained 7 years for compliance. Retrieval rare."

✅ Export to **S3 + Glacier Deep Archive** with **S3 Object Lock** in Compliance Mode
- Object Lock prevents deletion even by root
- Deep Archive at $0.001/GB-month

---

### Pattern 4: Shared Files for Auto Scaling Web Tier

> "Web fleet behind ALB. Files (uploads, themes) need to be shared across all instances."

✅ **EFS** mounted on all instances
- Auto-scales, multi-AZ
- All Linux EC2 see same files

---

### Pattern 5: HPC Training Data

> "10 TB of training images in S3. Need parallel access from 100 GPU instances."

✅ **FSx for Lustre linked to S3 bucket**
- Hundreds of GB/s parallel throughput
- Pay for the FSx only during training, delete after

---

### Pattern 6: Replace Tape Backups

> "Eliminate the on-prem tape library. Backup software still writes to tape."

✅ **AWS Storage Gateway - Tape Gateway**
- Virtual Tape Library (VTL) interface
- Virtual tapes auto-archived to Glacier

---

### Pattern 7: Move 500 TB to AWS in Limited Bandwidth

> "Limited 100 Mbps internet link. Need 500 TB in S3 within a month."

✅ **AWS Snowball Edge** (multiple devices in parallel)
- Online transfer: ~50 days theoretical, more in practice
- Snowball: 1-2 weeks logistics

---

### Pattern 8: Multi-Region Backup

> "Backups must survive region disaster."

✅ **S3 Cross-Region Replication (CRR)** to a DR region bucket
- Optional: enable Replication Time Control (RTC) for <15 min lag
- **AWS Backup** can orchestrate cross-region copies for RDS, EBS, EFS, etc.

---

## ⚡ The Master Storage Quick Recall Card

```
┌─────────────────────────────────────────────────┐
│ BY ACCESS PATTERN                                │
├─────────────────────────────────────────────────┤
│ One EC2 attaches block       → EBS               │
│ Many EC2 share files (NFS)   → EFS               │
│ Windows file shares          → FSx for Windows   │
│ HPC parallel filesystem      → FSx for Lustre    │
│ Multi-protocol enterprise    → FSx for ONTAP     │
│ Object storage / API         → S3                │
│ Backup, immutable            → S3 + Object Lock  │
│ Archive (rarely accessed)    → S3 Glacier tiers  │
├─────────────────────────────────────────────────┤
│ BY USE CASE                                      │
├─────────────────────────────────────────────────┤
│ Default for new workloads    → S3 + gp3 + EFS    │
│ Container shared volume      → EFS               │
│ ML training data             → S3 + FSx Lustre   │
│ Compliance archive 7+ yr     → Glacier Deep      │
│ Lift-shift file server       → FSx + DataSync    │
│ Hybrid block access          → Volume Gateway    │
│ Cloud-backed file gateway    → S3/FSx File GW    │
│ Replace tape library         → Tape Gateway      │
│ Mass offline transfer        → Snow Family       │
└─────────────────────────────────────────────────┘
```
