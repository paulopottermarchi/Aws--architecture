# High Availability & Disaster Recovery Architecture — AWS Multi-Cloud

> Architecture design project developed as part of a technical challenge for a Cloud/Data Engineering role.  
> Focus: high availability, disaster recovery, cost-efficient archival, and multi-cloud storage redundancy.

---

## Overview

This project proposes a cloud-native architecture on AWS with multi-cloud storage integration (Azure and GCP), designed to handle high availability at the application and database layers, asynchronous workload processing, and long-term data archival at low cost.

The architecture covers the full data flow: from API ingestion through load balancing, serverless processing, queue-based async compute, and tiered storage with automated lifecycle management.

---

## Architecture Diagram

                                   ┌───────────────┐
                                   │   Database    │
                                   │               │
                      ┌─────────────────┬─────────────────┐
                      │                 │                 │
                      │                 │                 │
                   ┌──▼──┐           ┌──▼──┐           ┌──▼──┐
                   │Multi-│          │Multi-│         │Cross-│
                   │  AZ  │          │  AZ  │         │Region│
                   │ Node │          │ Node │         │Replica
                   └──┬──┘           └──┬──┘           └──┬──┘
                      │                 │                 │
                      │                 │                 │
┌───────────┐      ┌─▼──┐           ┌──▼──┐            ┌──▼──┐
│API Gateway│      │Main│           │Worker│           │Worker│
└──────┬────┘      │Server          │Node 1│           │Node 2│
       │           └──┬──┘          └───┬─┘            └──┬──┘
       │              │                 │                 │
       │              │                 │                 │
       │          ┌──▼──┐           ┌───▼───┐         ┌───▼───┐
┌───────▼───────┐ │     │           │       │         │       │
│Load Balancer  │ │Queue│           │Worker │         │Worker │
└───────────────┘ │     │           │Node 3 │         │Node 4 │
                  └──┬──┘           └───┬───┘         └───┬───┘
                     │                  │                 │
                     │                  │                 │
                  ┌──▼──┐           ┌───▼───┐         ┌───▼───┐
                  │Lambda│          │       │         │       │
                  │Function         │Analytics        │VPC    │
                  └───┬─┘           └───────┘         │Subnet │
                      │                               └───────┘
                   ┌──▼──┐
                   │Data │
                   │Store│
                   └──┬──┘
                      │
           ┌──────────┴──────────┐
           │                     │ 
  ┌────────▼──────┐       ┌──────▼────────┐
  │Azure          │       │Google         │
  │Blob Storage   │       │Cloud Storage  │
  └───────────────┘       └───────────────┘
           │                         │
  ┌────────┴─────────────────────────┴──────────┐
  │                   Storage                   │
  │        ┌────────────┐          ┌───────────┐│
  │        │            │          │           ││
  └────────┤     S3     │──────────┤  Glacier  ││
           │            │          │           ││
           └────────────┘          └───────────┘│
                Lifecycle Policy                │
                         ───────────────────────┘


> Components follow AWS icon conventions. The diagram covers all layers: ingestion, compute, database, and storage.

---

## Components

### Application Layer

| Component | Role |
|---|---|
| **API Gateway** | Entry point for all external API requests — managed publishing, monitoring, and security |
| **Load Balancer** | Distributes incoming traffic across worker instances using Round Robin / Smart Load Balancing |
| **Lambda** | Serverless functions for event-driven and asynchronous task processing |
| **Main Server (EC2)** | Coordinates storage operations and connects to cloud storage providers |
| **Queue (SQS)** | Decouples the application layer; ensures ordered, resilient task delivery during demand spikes |
| **Worker Nodes (1–3)** | Parallel compute nodes pulling tasks from the queue for horizontal scaling |
| **VPC Subnet** | Private virtual network for secure, isolated internal communication |
| **Analytics** | Final processing layer for BI tooling and insight generation |

### Database Layer

| Component | Role |
|---|---|
| **Multi-AZ Deployment** | Synchronous data replication across Availability Zones in the primary region |
| **Cross-Region Replica** | Asynchronous replication to a secondary region for geographic disaster recovery |

> Database design prioritizes **Availability** in the CAP theorem tradeoff — the system continues operating in the presence of node failures, accepting eventual consistency at the cross-region replica level.

### Storage Layer

| Component | Role |
|---|---|
| **Amazon S3** | Durable object storage for active and intermediate data |
| **Amazon Glacier** | Low-cost archival for infrequently accessed data (long-term retention) |
| **Azure Blob Storage** | Cross-provider redundancy and backup |
| **Google Cloud Storage** | Additional multi-cloud resilience layer |

---

## Data Flow

```
External Request
    └──▶ API Gateway
             └──▶ Load Balancer (Round Robin / Smart LB)
                      └──▶ Lambda (serverless, event-driven)
                               └──▶ Main Server (EC2) ◀──▶ Data Layer
                                        ├──▶ Azure Blob Storage (backup)
                                        ├──▶ Google Cloud Storage (backup)
                                        └──▶ Queue (SQS)
                                                 └──▶ Worker Nodes 1, 2, 3 (parallel)
                                                          └──▶ S3 (intermediate storage)
                                                          └──▶ Analytics (insights)
```

---

## Database Design — CAP Theorem Considerations

The database layer was designed with explicit awareness of the CAP theorem tradeoffs:

- **Multi-AZ (CA)**: prioritizes Consistency + Availability within a region. Synchronous replication ensures all nodes have the same data view; automatic failover handles AZ-level outages.
- **Cross-Region Replica (AP)**: prioritizes Availability + Partition Tolerance across regions. Asynchronous replication accepts a small lag in exchange for geographic resilience.

This dual approach covers both operational HA (Multi-AZ) and catastrophic DR (Cross-Region) without forcing a single global CAP tradeoff.

---

## Disaster Recovery Strategy

### Failover Procedure
1. Detect failure in primary region (CloudWatch alarms / health checks)
2. Promote Cross-Region Replica to primary
3. Redirect traffic via Route 53 DNS failover
4. Validate data consistency across all nodes
5. Restore services and monitor stability
6. Execute failback to primary region once confirmed healthy

> **Best practice:** Test failover periodically (quarterly minimum) to validate actual RTO/RPO targets — not just theoretical ones.

---

## Data Archival Strategy

### S3 Lifecycle Policy

A Lifecycle Policy on Amazon S3 automatically transitions objects to Glacier after a defined retention period:

```
S3 (active data, high performance)
    └──[90 days of inactivity]──▶ Glacier (archive, low cost)
```

### Glacier Recovery Options

| Tier | Time | Use Case |
|---|---|---|
| Expedited | 1–5 min | Urgent, time-sensitive recovery |
| Standard | 3–5 hours | Typical planned recovery |
| Bulk | 5–12 hours | Large-scale, cost-optimized recovery |

### Recovery Procedure
1. Identify objects to restore (vault + object keys)
2. Submit restore request via AWS Console or CLI
3. Select appropriate retrieval tier
4. Data becomes temporarily available in a target S3 bucket
5. Access, download, or pipeline the restored data as needed
6. Monitor retrieval costs (billed per GB + per request)

---

## Multi-Cloud Design

Storage redundancy spans three cloud providers to eliminate vendor lock-in and enable cross-cloud failover:

| Provider | Service | Role |
|---|---|---|
| AWS | S3 + Glacier | Primary storage, archival, and lifecycle management |
| Microsoft Azure | Blob Storage | Cross-provider redundancy and geographic backup |
| Google Cloud | Cloud Storage | Additional resilience and backup layer |

---

## Architecture Decisions — Justification

| Decision | Justification |
|---|---|
| Multiple Worker Nodes | Horizontal scaling + parallel task processing; individual node failure doesn't stop the pipeline |
| Queue-based decoupling | Absorbs demand spikes without cascading failures; workers process at their own pace |
| Multi-AZ Database | Eliminates single points of failure within a region; automatic failover with minimal downtime |
| Cross-Region Replica | Covers region-level catastrophic failures; RPO determined by replication lag |
| S3 → Glacier Lifecycle | Automated cost optimization; removes manual intervention from archival process |
| Multi-cloud Storage | Prevents vendor lock-in at the storage layer; increases availability in extreme failure scenarios |
| Lambda for async tasks | Serverless execution eliminates idle resource costs; scales automatically with event volume |
| VPC Subnet | Network isolation ensures internal services are not exposed to the public internet |

---

## Tech Stack

- **Primary Cloud**: AWS
- **Storage (multi-cloud)**: Azure Blob Storage, Google Cloud Storage
- **Compute**: EC2 (Main Server + Worker Nodes), AWS Lambda
- **Storage (AWS)**: Amazon S3, Amazon Glacier
- **Networking**: Amazon API Gateway, Elastic Load Balancer, Amazon VPC, Amazon SQS
- **Database**: Amazon RDS — Multi-AZ + Cross-Region Read Replica

---

## Project Status

> This is an **architecture design and documentation project**.  
> Components were designed, documented, and presented — not deployed to a live environment.  
> The scope covers architectural decision-making, DR planning, CAP theorem tradeoff analysis, and cost strategy.

---

*Designed and documented by Paulo Ricardo Potter*
