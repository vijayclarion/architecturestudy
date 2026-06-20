# AWS Solutions Architect Professional (SAP-C02) — Study Notes

> Comprehensive notes built from the Digital Cloud Training course (905 slides) following the **Architect's 5-Layer Learning System**: cheat sheets, diagrams, decision trees, real anchors, and active recall.

---

## 🎯 The Exam at a Glance

```
┌─────────────────────────────────────────────────┐
│  SAP-C02 — AWS Solutions Architect Professional │
├─────────────────────────────────────────────────┤
│  Duration:        180 minutes                    │
│  Questions:       75                             │
│  Cost:            $300 USD                       │
│  Passing Score:   750 / 1000                     │
│  Format:          Multiple choice + Multi-response│
│  Delivery:        Testing center or online       │
└─────────────────────────────────────────────────┘
```

### Domain Weights

| Domain | Topic | Weight |
|---|---|---|
| **1** | Design Solutions for Organizational Complexity | 26% |
| **2** | Design for New Solutions | 29% |
| **3** | Continuous Improvement for Existing Solutions | 25% |
| **4** | Accelerate Workload Migration and Modernization | 20% |

---

## 📁 Folder Structure

```
aws-sap-prep/
├── 01-Foundation/              Accounts, Organizations, IAM, Federation
├── 02-Networking/              VPC, Hybrid Connectivity, DNS, CDN
├── 03-Compute-Storage/         EC2, Auto Scaling, ELB, S3, EBS, EFS, FSx
├── 04-Databases-Caching/       RDS, Aurora, DynamoDB, ElastiCache, Redshift
├── 05-Serverless-Containers/   Lambda, API GW, ECS, EKS, Fargate, Step Fn
├── 06-Deployment-Migration/    CloudFormation, CI/CD, DataSync, DMS, SMS
├── 07-Analytics-Monitoring/    Kinesis, Athena, EMR, Glue, CloudWatch, X-Ray
├── 08-Security/                KMS, Secrets Mgr, GuardDuty, Shield, WAF
├── 99-Cheat-Sheets/            One-page summaries & decision trees
└── diagrams/                   Mermaid source files
```

---

## 🗺️ Study Plan — 6 Week Roadmap

```
Week 1 → Foundation
        IAM, Organizations, SCPs, Federation
        
Week 2 → Networking Deep Dive
        VPC peering, Transit Gateway, Direct Connect,
        Site-to-Site VPN, PrivateLink, Route 53
        
Week 3 → Compute + Storage
        EC2 placement, Auto Scaling, ALB/NLB,
        S3 storage classes, EBS types, FSx variants
        
Week 4 → Databases + Caching + Analytics
        Aurora Global, DynamoDB Global Tables,
        ElastiCache, Redshift, Kinesis
        
Week 5 → Serverless + Containers + DR
        Lambda patterns, ECS vs EKS, Step Functions,
        DR strategies (Backup→Pilot→Warm→Multi-site)
        
Week 6 → Security, Migration, Cost Optimization
        KMS, GuardDuty, Migration Hub, 7 R's,
        Mock exams + review weak areas
```

---

## 🧠 The 5-Layer Learning Method (Applied to AWS)

### Layer 1 — One Page Cheat Sheets
Each major service has a one-page cheat sheet in `99-Cheat-Sheets/`

### Layer 2 — Diagrams as Code (Mermaid)
All architecture flows are in `diagrams/` as `.mermaid` source files.

### Layer 3 — Decision Trees
"When to use what" is the SAP-C02 superpower:
- ALB vs NLB vs GWLB
- DynamoDB vs Aurora vs RDS
- Direct Connect vs VPN vs Transit Gateway
- Backup vs Pilot Light vs Warm Standby vs Multi-Site

### Layer 4 — Anchor to Real Companies
| Pattern | Real-World Example |
|---|---|
| Multi-Region Active-Active | Netflix using Route 53 + DynamoDB Global Tables |
| Event-driven Saga | Uber order pipeline (SNS → SQS → Lambda → Step Fn) |
| Data Lake | Airbnb (S3 + Athena + Glue + Lake Formation) |
| Hybrid burst | Capital One (Direct Connect + Transit Gateway) |
| Disaster Recovery | Expedia (Route 53 failover + warm standby) |

### Layer 5 — Active Recall
- Whiteboard 1 architecture every weekend
- Anki flashcards for trade-off questions
- Explain out loud before reading the answer

---

## 📚 How to Use This Repo

1. **Read** the section README first (high-level map)
2. **Study** each concept note using the **fixed template**:

```markdown
## Problem It Solves
## How It Works (with diagram)
## When to Use
## When NOT to Use
## Trade-offs
## Real World Example
## Interview / Exam Soundbite
```

3. **Recall** by closing the file and explaining out loud
4. **Test** with the decision trees in `99-Cheat-Sheets/`
5. **Practice** with the scenario-based questions at the end of each section

---

## ✅ Section Index

| # | Section | Status | Exam Weight |
|---|---|---|---|
| 01 | [Foundation — IAM, Orgs, Federation](./01-Foundation/README.md) | 🔥 Critical | ⭐⭐⭐⭐⭐ |
| 02 | [Networking — VPC, Hybrid, DNS](./02-Networking/README.md) | 🔥 Critical | ⭐⭐⭐⭐⭐ |
| 03 | [Compute & Storage](./03-Compute-Storage/README.md) | 🔥 Critical | ⭐⭐⭐⭐⭐ |
| 04 | [Databases & Caching](./04-Databases-Caching/README.md) | High | ⭐⭐⭐⭐ |
| 05 | [Serverless & Containers](./05-Serverless-Containers/README.md) | High | ⭐⭐⭐⭐ |
| 06 | [Deployment & Migration](./06-Deployment-Migration/README.md) | High | ⭐⭐⭐⭐ |
| 07 | [Analytics & Monitoring](./07-Analytics-Monitoring/README.md) | Medium | ⭐⭐⭐ |
| 08 | [Security](./08-Security/README.md) | High | ⭐⭐⭐⭐ |
| 99 | [Cheat Sheets & Decision Trees](./99-Cheat-Sheets/README.md) | 🔥 Critical | ⭐⭐⭐⭐⭐ |

---

> ✨ **Pro Tip:** SAP-C02 doesn't test if you know what a service does. It tests if you know **which service to pick** when given 5 valid-sounding options. Focus on the **trade-offs** in every note.
