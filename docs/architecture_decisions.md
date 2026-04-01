# Architecture Decisions — High Availability & Disaster Recovery on AWS

This document details the reasoning behind each architectural choice made in this project. The goal is not just to describe *what* was built, but *why* each decision was made and what tradeoffs were accepted.

---

## 1. API Gateway as the Single Entry Point

**Decision:** All external requests enter through Amazon API Gateway before reaching any internal service.

**Reasoning:**
- Centralizes authentication, rate limiting, and request validation in one managed layer
- Decouples external consumers from internal service topology — internal changes don't affect API contracts
- Provides built-in throttling to protect downstream services from traffic surges
- Enables versioning, allowing breaking changes without disrupting existing consumers

**Tradeoff accepted:** Adds a network hop and marginal latency. Acceptable given the security and operational benefits at scale.

---

## 2. Load Balancer with Round Robin + Smart Load Balancing

**Decision:** Elastic Load Balancer distributes traffic across Worker Nodes using both Round Robin and health-aware routing.

**Reasoning:**
- Round Robin ensures even distribution under normal conditions
- Smart/health-aware routing automatically removes unhealthy nodes from the pool without manual intervention
- Enables zero-downtime deployments via rolling updates — new nodes join the pool before old ones are removed

**Tradeoff accepted:** ELB adds cost. For low-traffic workloads, a single server might be cheaper. This design is optimized for availability, not minimum cost.

---

## 3. Lambda for Asynchronous Processing

**Decision:** AWS Lambda handles event-driven and asynchronous tasks triggered by the API Gateway.

**Reasoning:**
- Serverless execution eliminates idle resource costs — you pay only for actual invocations
- Scales automatically with event volume without capacity planning
- Suitable for short-lived, stateless operations (data transformation, notification dispatch, lightweight ETL triggers)

**Tradeoff accepted:** Lambda has execution time limits (15 minutes max) and cold start latency. Long-running or stateful workloads belong on EC2, not Lambda.

---

## 4. SQS Queue for Application Decoupling

**Decision:** A queue sits between the application layer and the compute nodes, mediating all task delivery.

**Reasoning:**
- Decoupling absorbs demand spikes: if workers are busy, messages wait in the queue rather than being dropped or causing failures upstream
- Enables independent scaling of producers and consumers — the API layer and compute layer scale on different metrics
- Provides at-least-once delivery guarantees, reducing the risk of lost tasks during transient failures
- Simplifies retry logic: failed messages can be requeued without application-level retry code

**Tradeoff accepted:** Introduces asynchrony — results are not immediately available after a request. Systems that require synchronous responses need a different pattern (e.g., request/response via Lambda + API Gateway directly).

---

## 5. EC2 Main Server for Storage Coordination

**Decision:** A persistent EC2 instance acts as the Main Server coordinating operations between the data layer and cloud storage providers.

**Reasoning:**
- Long-running coordination tasks (syncing to Azure Blob, GCS, managing S3 writes) are not suitable for Lambda given execution limits
- EC2 provides persistent state, filesystem access, and stable network connections required for large data transfers
- Allows fine-grained control over retry logic, error handling, and transfer monitoring

**Tradeoff accepted:** EC2 is always-on cost. Right-sizing the instance type and using Reserved Instances mitigates this for predictable workloads.

---

## 6. Multi-AZ Database Deployment

**Decision:** The primary database runs in a Multi-AZ configuration with synchronous replication across Availability Zones.

**Reasoning:**
- Eliminates single points of failure within the primary region
- Automatic failover to the standby node requires no manual intervention and completes in typically under 2 minutes
- Synchronous replication guarantees zero data loss (RPO ≈ 0) for AZ-level failures
- AWS manages the replication, failover, and DNS update transparently

**CAP Theorem position:** This is a CA configuration — prioritizing Consistency and Availability within the region. Partition Tolerance at the network level is handled by AWS infrastructure.

**Tradeoff accepted:** The standby node is not readable — it exists solely for failover. Read replicas would be needed to offload read traffic.

---

## 7. Cross-Region Read Replica for Disaster Recovery

**Decision:** An asynchronous read replica runs in a secondary AWS region, serving as the DR target.

**Reasoning:**
- Protects against region-level failures (natural disasters, large-scale outages) that Multi-AZ cannot cover
- Asynchronous replication avoids write latency penalties on the primary — cross-region synchronous replication would significantly impact performance
- In a DR event, the replica can be promoted to primary and traffic redirected via Route 53

**CAP Theorem position:** This is an AP configuration — prioritizing Availability and Partition Tolerance at the cost of potential replication lag (eventual consistency). The replica may be slightly behind the primary at the moment of failover.

**Tradeoff accepted:** RPO is not zero — data written after the last replication cycle may be lost in a catastrophic failure. This is an accepted tradeoff for geographic DR. The RPO window depends on replication lag, typically seconds to minutes.

---

## 8. Amazon S3 as Primary Object Storage

**Decision:** S3 stores all active and intermediate data produced by the compute pipeline.

**Reasoning:**
- 11 nines of durability (99.999999999%) — effectively eliminates data loss risk from storage failures
- Decoupled from compute — data persists independently of worker node lifecycle
- Native integration with Glacier for lifecycle management, and with AWS analytics services (Athena, EMR, Glue)
- Pay-per-use pricing scales linearly with actual storage consumed

---

## 9. S3 Lifecycle Policy → Glacier

**Decision:** Objects inactive for 90 days are automatically transitioned to Amazon Glacier via a Lifecycle Policy.

**Reasoning:**
- Glacier costs significantly less per GB than S3 Standard
- Lifecycle Policy automates the transition — no manual intervention, no risk of forgetting to archive old data
- 90-day threshold chosen as a balance between keeping frequently referenced data accessible and minimizing storage costs for cold data
- Glacier maintains the same durability guarantees as S3

**Tradeoff accepted:** Glacier retrieval is not immediate. Standard retrieval takes 3–5 hours; expedited retrieval (minutes) costs more. This is acceptable for archival data that is not operationally critical.

---

## 10. Multi-Cloud Storage (Azure Blob + Google Cloud Storage)

**Decision:** Azure Blob Storage and Google Cloud Storage are integrated as additional backup layers alongside AWS S3.

**Reasoning:**
- Eliminates vendor lock-in at the storage layer — the most critical layer for long-term data retention
- Provides geographic redundancy across cloud providers, not just AWS regions
- Enables cross-cloud failover in extreme scenarios where AWS itself has a broad outage
- Marginal cost at the storage layer is low relative to the resilience benefit

**Tradeoff accepted:** Multi-cloud introduces operational complexity — egress costs, additional IAM/credential management, and synchronization overhead. This tradeoff is justified for data that must survive any single provider failure.

---

## 11. VPC Subnet for Network Isolation

**Decision:** Internal resources (Main Server, Worker Nodes, database) run inside a private VPC Subnet.

**Reasoning:**
- Internal services are not exposed to the public internet — attack surface is minimized
- Traffic between internal components stays within AWS's private network, reducing latency and egress costs
- Security Groups and NACLs provide fine-grained inbound/outbound traffic control
- Required for compliance in regulated industries (financial services, healthcare)

---

## Summary of Key Tradeoffs

| Decision | What was prioritized | What was accepted |
|---|---|---|
| API Gateway | Security, decoupling | Marginal latency |
| SQS Queue | Resilience, async | No synchronous response |
| Multi-AZ DB | RPO ≈ 0, auto-failover | Standby not readable |
| Cross-Region Replica | Geographic DR | RPO > 0 (replication lag) |
| Glacier Lifecycle | Cost optimization | Slow retrieval (hours) |
| Multi-cloud Storage | Vendor independence | Operational complexity |
| Lambda | Cost efficiency | 15-min execution limit |
