# WDP-NFRS.md
**World Dispute Platform — Non-Functional Requirements**
*Version: 1.0 | Extracted: April 2026 | Source: WDP Documentation Suite v2.0*

---

## How to Read This Document

NFRs are organised into five sections: Performance, Availability & Resilience, Security & Compliance, Scalability, and Operational Constraints. Within each section, requirements are grouped by stage (Stage 1 production, Stage 2 production, Stage 3 planned) and flagged where necessary.

**Flags used:**
- ⚠️ OUTDATED — present in source docs but superseded or inconsistent with later evidence; retain for review
- ⚠️ VERIFY — target appears in planning docs only and has not been validated in production

All Stage 3 targets are inherently ⚠️ VERIFY — they originate from a planning scaffold that has not been built or measured.

---

## 1. Performance

### 1.1 End-to-End File Processing (Stage 1)

| Requirement | Target | Percentile | Measurement Window | SLO Breach Alert |
|---|---|---|---|---|
| File processing end-to-end latency | < 10 minutes | P95 | Rolling 1 hour | > 15 minutes |
| Outbox publish lag (write → Kafka) | < 2 minutes | P95 | Rolling 5 minutes | > 5 minutes (CRITICAL) |
| File upload → outbox insert | < 30 seconds | Target | Per file | — |
| File upload → outbox insert | < 45 seconds | P95 | Per file | — |
| File upload → outbox insert | < 60 seconds | P99 | Per file | — |
| Outbox insert → Kafka publish | < 2 minutes | Target | Per batch | — |
| Outbox insert → Kafka publish | < 3 minutes | P95 | Per batch | — |
| Outbox insert → Kafka publish | < 5 minutes | P99 | Per batch | — |
| Kafka publish → evidence attached | < 30 seconds | Target | Per row | — |
| Kafka publish → evidence attached | < 60 seconds | P95 | Per row | — |
| Kafka publish → evidence attached | < 120 seconds | P99 | Per row | — |
| Full evidence flow end-to-end (upload → attached) | < 3 minutes | Target | — | — |
| Full evidence flow end-to-end (upload → attached) | < 5 minutes | P95 | — | — |
| Full evidence flow end-to-end (upload → attached) | < 10 minutes | P99 | — | — |

⚠️ OUTDATED — An earlier document (WDP-Final-Architecture-Document-v2.0.md) states ACK generation target of < 60 seconds at P99. The production SLO document (Cross-Cutting--WDP-Monitoring-Standards.md) does not repeat this figure and instead defines ACK generation success rate (> 95%) as the primary SLO. Verify which metric governs ACK SLA in production.

### 1.2 Throughput (Stage 1)

| Requirement | Target | Notes |
|---|---|---|
| Platform throughput | 10,000 disputes / hour | Design target; load test pending as of Oct 2025 |
| File Processor row throughput | 10,000 rows / minute | Per instance |
| Publisher Scheduler throughput | 3,000 rows / minute | 100 rows/batch × 30 batches/min |
| Evidence Worker throughput | 1,000 rows / minute | 10 concurrent consumers × 100 rows/min |
| Document Management API batch capacity | 10 files / request | Maximum batch size per upstream API call |
| Kafka message capacity | 50,000 messages / second | Current MSK provisioned headroom; ceiling ~100,000 |
| Aurora PostgreSQL transaction capacity | 10,000 TPS | Multi-AZ cluster, auto-scaling read replicas |

⚠️ OUTDATED — WDP-Final-Architecture-Document-v2.0.md references a File Processor capable of scaling to 350 pods. This was the Stage 1.4 Unified Outbox design. Current Stage 1 production auto-scaling caps the File Processor at 10 instances to prevent S3 throttling. The 350-pod figure should be treated as obsolete.

### 1.3 API Response Latency (Stage 1 & Stage 2)

| Requirement | Target | Percentile | Source |
|---|---|---|---|
| Internal API response time | < 200 ms | P95 | Cross-cutting target |
| Notification delivery from case state change | < 30 seconds | — | Stage 1 design target |
| Encryption API — encrypt | < 50 ms | P95 | Excl. rare DEK refresh |
| Encryption API — decrypt | < 75 ms | P95 | Incl. KMS call when DEK cached |
| Merchant portal — queue list load | < 1 second | — | Stage 2 UI target |
| Merchant portal — queue case list load | < 2 seconds | — | Stage 2 UI target |
| Merchant portal — dispute detail load | < 2 seconds | — | Stage 2 UI target |
| Merchant portal — queue refresh | < 500 ms | — | Stage 2 UI target |
| Dispute list — page load | < 5 seconds | P95 | Stage 2 UI alert threshold |
| Dispute list — search API | < 10 seconds | P95 | Stage 2 UI alert threshold |
| BRE validation per step | < step-specific timeout | P95 | Steps have individual timeout thresholds; breach triggers WARNING |

### 1.4 Circuit Breaker Recovery

| Requirement | Target |
|---|---|
| Circuit breaker recovery time | < 5 minutes per incident |
| Circuit breaker recovery — alert threshold | > 10 minutes (WARNING) |

### 1.5 Stage 3 Analytics Performance ⚠️ VERIFY

All Stage 3 targets are from planning documents only. None have been measured.

| Requirement | Target | Percentile |
|---|---|---|
| ETL job completion (daily load) | < 2 hours | For 1M cases/day |
| ETL data freshness | < 24 hours | P95 |
| ETL failure rate | < 1% | — |
| Simple query (pre-built reports) | < 2 seconds | P95 |
| Complex query (custom reports) | < 10 seconds | P95 |
| Dashboard load time | < 3 seconds | P95 |
| ML prediction latency | < 100 ms | P95 |
| Report API response | < 500 ms | P95 |
| Data export generation (10k rows) | < 30 seconds | — |
| Concurrent analytics users supported | 1,000+ | — |

---

## 2. Availability and Resilience

### 2.1 Service Availability Targets

| Component | Availability Target | Notes |
|---|---|---|
| Encryption API | 99.9% | Three-nines SLA |
| Publisher Scheduler | > 99.9% | Measured monthly |
| Aurora PostgreSQL | 99.99% | Multi-AZ; auto-failover |
| Kafka (AWS MSK) | 99.9% | 3-broker, 3-AZ cluster |
| S3 Storage | 99.99% | AWS managed |
| EKS Cluster | 99.95% | Auto-scaling, multi-AZ |
| Stage 2 overall (portal + backend) | 99.9% | Design target |
| Stage 3 analytics services | > 95% | ⚠️ VERIFY — planning target only |

### 2.2 Recovery Time and Recovery Point Objectives

| Failure Scenario | RTO | RPO | Recovery Mechanism |
|---|---|---|---|
| Pod failure | 30 seconds | 0 | Kubernetes self-healing |
| Node failure | 2 minutes | 0 | Auto-scaling, pod rescheduling |
| Availability zone failure | 5 minutes | 0 | Multi-AZ deployment |
| Database corruption | 30 minutes | 5 minutes | Aurora PITR from automated backups |
| Region failure (full DR) | 1 hour | 5 minutes | Aurora Global Database failover to us-west-2 |
| Complete data centre loss | 4 hours | 15 minutes | Cross-region DR site |

⚠️ OUTDATED — WDP-Final-Architecture-Document-v2.0.md states RTO = 4 hours and RPO = 5 minutes as general targets. The Infrastructure Deployment Consolidated document provides a more granular breakdown (above) superseding the general figure. The 4-hour / 5-minute combination should only be read as the worst-case full-region-failure scenario, not as the typical RTO.

⚠️ OUTDATED — Evidence file processing design (Stage 1.7) states RTO = 1 hour, RPO = 5 minutes for that component specifically. This conflicts with the infrastructure document's full-region RTO of 1 hour. The 1-hour evidence component figure appears to refer to application recovery (restarting services), not full DR. Verify which is the governing DR SLA.

### 2.3 Resilience Behaviour

| Requirement | Specification |
|---|---|
| Document Management API circuit breaker — failure threshold | 20% failure rate triggers OPEN (per-merchant) |
| Document Management API circuit breaker — wait duration | 60 seconds in OPEN state |
| Document Management API circuit breaker — half-open test calls | 10 calls before closing |
| Network integration circuit breakers — failure threshold | 50% failure rate triggers OPEN |
| Network integration circuit breakers — wait duration | 30 seconds in OPEN state |
| Encryption API circuit breaker — scope | Global (not per-merchant) |
| DEK cache window on KMS outage | 6 hours of continued operation before forced failure |
| Kafka consumer — delivery guarantee | At-least-once; offset committed only after full processing |
| Outbox retry policy | At least 2 retry attempts for transient failures before terminal ERROR |
| Notification channel isolation | Per-channel outbox; failure in one channel does not affect others |
| Merchant isolation | Per-merchant circuit breakers and Kafka partitions; one merchant failure cannot cascade to others |
| ACK generation success rate SLO | > 95% | Rolling 1 hour; alert at < 90% (CRITICAL) |
| BRE crash recovery success rate | 100% (verified across 50+ crash scenarios in testing) |

### 2.4 Kafka Durability

| Requirement | Specification |
|---|---|
| Replication factor | 3 brokers across 3 availability zones |
| Minimum in-sync replicas | 2 (data not acknowledged unless written to 2 replicas) |
| Unclean leader election | Disabled (prevents data loss on leader election) |
| Producer acknowledgment mode | All in-sync replicas must acknowledge (acks=all) |
| Producer idempotence | Enabled (prevents duplicate messages on producer retry) |

---

## 3. Security and Compliance

### 3.1 Compliance Frameworks

| Framework | Scope | Status |
|---|---|---|
| PCI-DSS 3.2.1 | Full platform — all cardholder data flows | ✅ Active |
| SOC 2 Type II | Security, availability, processing integrity, confidentiality, privacy | ✅ Active (annual audit) |
| GDPR | Data subject rights, data privacy, right to erasure | ✅ Active |
| CCPA | California resident data rights, deletion within 90 days of request | ✅ Active |
| SOX | Immutable audit trail for financial communications (ACK snapshots) | ✅ Active |

### 3.2 PAN Handling Constraints

| Requirement | Specification |
|---|---|
| Plaintext PAN scope | Only the Encryption API may handle plaintext PAN; no other component may store or process it |
| PAN at rest | Must not be stored in plaintext anywhere in the system |
| PAN in transit | Must not travel in plaintext across any network boundary or through Kafka |
| PAN in logs | Must never appear in application logs, metrics, or traces |
| PAN in ACK files | ACK files must use merchant-provided chargeback identifier, not PAN |
| Encryption algorithm | AES-256-GCM |
| Tokenisation algorithm | HMAC-SHA256 (for HPAN — deterministic, non-reversible) |
| Key storage | AWS KMS with FIPS 140-2 Level 3 validated HSMs; keys never leave HSM |

### 3.3 Key Management Constraints

| Requirement | Specification |
|---|---|
| Customer Master Key (CMK) rotation | Automatic, every 30 days |
| Data Encryption Key (DEK) lifetime | 6 hours maximum in memory |
| DEK at rest | Stored only as KMS ciphertext (wrapped by CMK) |
| HMAC key storage | AWS Secrets Manager |
| Key management provider | AWS KMS — no alternative fallback key path exists |
| Evidence files at rest | SSE-KMS encryption in S3; keys rotated every 90 days |
| ACK files at rest | S3 SSE-KMS with dedicated CMK |
| All data in transit | TLS 1.3 minimum; TLS 1.2 for legacy integrations where TLS 1.3 unavailable |

### 3.4 Access Control

| Requirement | Specification |
|---|---|
| Service-to-service authentication | OAuth 2.0 client credentials (JWT) plus mutual TLS (mTLS) |
| PAN decrypt scope — full | Restricted to Chargeback Worker only |
| PAN decrypt scope — masked | Available to Evidence Worker (first 6 + last 4 digits only) |
| PAN encrypt scope | File Processor and API Processor only |
| User authentication | JWT via shared IDP; OAuth 2.0 |
| JWT validation | Public key with `kid` claim (Phase 1); JWKS URL (Phase 2 — planned) |
| Merchant data isolation | Row-level security by merchant_id enforced at API and database levels |
| Merchant API — additional auth layer | API Key required in addition to OAuth JWT for external merchants |
| Merchant API — rate limit | 1,000 requests per hour per merchant |
| Operations user roles | Tier 1 Agent, Tier 2 Agent, Team Lead, Manager, Admin — with distinct action permissions |
| Queue access | Users may only access disputes in queues assigned to their role |

### 3.5 Audit and Retention

| Requirement | Specification |
|---|---|
| Audit log retention — primary | 7 years (PCI-DSS, SOC 2, SOX requirement) |
| Audit log hot storage | PostgreSQL, partitioned (2 years) |
| Audit log cold storage | AWS S3 Glacier after 2 years |
| ACK snapshot retention | All versions retained for 7 years (SOX immutability requirement) |
| KMS CloudTrail logs | 7 years in S3 (encrypted), Glacier after 1 year |
| Application logs | 1 year in CloudWatch; archived to S3 after 90 days |
| S3 evidence files | Versioning enabled; lifecycle policy: delete after 7 years |
| Audit trail properties | Append-only, immutable, partitioned; all state transitions attributed to component or user |
| GDPR right to erasure | PAN deleted from pan_store within 30 days of request; crypto_audit pseudonymised (HPAN nulled); audit record itself retained under legal obligation |
| GDPR right to access | Available via admin API query within 30 days |
| CCPA deletion | Available via admin API within 90 days of request |

### 3.6 Stage 3 Security Constraints ⚠️ VERIFY

| Requirement | Specification |
|---|---|
| PAN in analytics | Raw PAN must never appear in analytics; HPAN tokens only |
| Redshift row-level security | Merchants may only query rows belonging to their merchant_id |
| ML model access | Authenticated API; model versioning and audit trail required |
| Analytics data lineage | Source-to-report lineage tracked via metadata catalogue |
| Stage 3 audit trail retention | 7 years (aligning with platform standard) |

---

## 4. Scalability

### 4.1 Horizontal Scaling Boundaries

| Component | Min Instances | Max Instances | Scale Trigger |
|---|---|---|---|
| File Processor | 3 | 10 | CPU > 70% or memory > 80% |
| Publisher Scheduler | 2 | 2 (fixed) | Leader election; no further horizontal scaling |
| Chargeback Worker | — | — | Kafka consumer lag > 1,000 per pod |
| Evidence Worker | 5 | 20 | Kafka consumer lag > 1,000; max capped by Kafka partition count |
| EKS cluster nodes | 10 | 50 | CPU/memory utilisation; auto-scaling |

**Publisher Scheduler constraint:** The Publisher Scheduler uses a database-level advisory lock for single-writer guarantees. Only one instance is active as writer at any time; the second instance is a hot standby for failover only. This component cannot be horizontally scaled for throughput.

### 4.2 Kafka Partition Constraints

| Constraint | Value | Implication |
|---|---|---|
| wdp.file.outbox.events — partition count | 6 | Maximum effective parallel Evidence Worker instances = 6 without repartitioning |
| Partition key | merchant_id | All events for a merchant go to the same partition; partition skew expected |
| Top-5 merchant volume share | ~40% of total | These merchants create hot partitions; monitor lag on high-volume partitions |
| Partition count change | Requires topic recreation or manual rebalancing | Treat partition count as a semi-permanent decision |

⚠️ VERIFY — An archived document (v3.0) refers to 12 partitions for the evidence topic and 50 partitions for the file outbox topic. The production infrastructure document specifies 6 partitions for wdp.file.outbox.events and 3 for notification topics. The 50-partition figure may be an aspirational target from early design. Confirm actual production partition counts before planning scaling.

### 4.3 Database Scalability

| Requirement | Specification |
|---|---|
| Aurora read replicas — minimum | 2 (production) |
| Aurora read replicas — auto-scale maximum | 5 |
| Auto-scale trigger | CPU > 70% or connection count > 700 |
| Scale-in cooldown | 5 minutes |
| Scale-out cooldown | 60 seconds |
| Connection pool maximum per service | 20 connections (HikariCP) |

### 4.4 MSK Storage Constraint — Permanent Floor

MSK provisioned storage is **one-directional**: once storage is scaled up, it cannot be reduced. Every storage scaling event sets a new permanent floor. This constraint applies across all environments.

**Current provisioned storage:** 500 GB per broker (3 brokers = 1,500 GB total).

All future capacity planning decisions must treat any storage increase as an irreversible commitment. The recommended utilisation ceiling before scaling is 60% to maintain headroom.

### 4.5 Stage 3 Scalability ⚠️ VERIFY

| Requirement | Specification |
|---|---|
| Analytics concurrent user target | 1,000+ simultaneous users |
| ETL daily volume target | 1,000,000+ cases per day |
| S3 data lake growth | ~100 TB projected over time (rough estimate) |
| Redshift cluster initial sizing | Undetermined — open question as of November 2025 |
| ML inference scaling | Independent containerised microservice, GPU-capable instances for inference |

---

## 5. Operational Constraints

### 5.1 Deployment Windows

| Environment | Deployment Frequency | Approval | Rollback SLA |
|---|---|---|---|
| Development | On every commit (automated) | Not required | 5 minutes |
| Staging | Daily (automated) | Not required | 10 minutes |
| Production | Weekly (manual trigger) | Architect or tech lead approval required | 15 minutes |

**Standard production window:** Tuesday and Thursday, 10:00 AM – 2:00 PM ET (low-traffic period).

**Emergency deployments:** Permitted 24/7 with on-call engineer approval.

**Blackout period:** End of month is restricted due to chargeback volume spikes. No production deployments during this window except P0 emergency patches.

### 5.2 Incident Response Constraints

| Requirement | Target |
|---|---|
| On-call alert acknowledgement | Within 5 minutes of PagerDuty page |
| CRITICAL alerts | Page on-call engineer immediately |
| WARNING alerts | Deliver to Slack channel; no immediate page |
| Post-incident review | Required for all P0 and P1 incidents |
| Blast radius determination | Required as first step of triage — single merchant vs. platform-wide |

### 5.3 Data Retention and Archival

| Data Category | Active Retention | Archival Trigger | Archive Destination |
|---|---|---|---|
| Outbox rows (SUCCESS) | 90 days | After 90 days | Archived table; purged after 7 years |
| Audit logs (hot) | 2 years | After 2 years | S3 Glacier |
| All audit logs | 7 years total | — | Compliance requirement |
| ACK snapshots | 7 years | After 2 years | S3 Glacier |
| S3 evidence files | 7 years | After 2 years | S3 Glacier lifecycle |
| Application logs | 1 year (CloudWatch) | After 90 days | S3 archival |
| Redshift raw facts (Stage 3) | 2 years rolling | After 2 years | S3 Glacier |
| Redshift aggregates (Stage 3) | 7 years | — | Compliance requirement |

### 5.4 Monitoring Coverage Requirements

| Requirement | Specification |
|---|---|
| Metrics defined (Stage 1) | 88 named metrics with descriptions, types, and labels |
| Alerts configured (Stage 1) | 27 alerts with severity, threshold, and runbook reference |
| Grafana dashboards (Stage 1) | 7 production dashboards |
| Alert severity — CRITICAL | Fires PagerDuty page; requires acknowledgement within 5 minutes |
| Alert severity — WARNING | Fires Slack notification; no page |
| Observability approach | Golden signals: Latency, Traffic, Errors, Saturation |
| Latency instrumentation | Histograms required (not averages); P50, P95, P99 recorded |
| High-cardinality labels | Prohibited as metric labels (e.g., user_id, transaction_id) |
| SLO lines | Required on all latency graphs as visual threshold markers |

### 5.5 Backup and Verification

| Requirement | Frequency | Pass Criteria |
|---|---|---|
| Aurora database PITR test | Weekly | Restore completed within 30 minutes; data integrity verified |
| Application config backup verification | Weekly | ConfigMaps and secrets restored successfully |
| Full DR drill (failover to us-west-2) | Quarterly | Full failover completed within 1 hour |
| Annual data loss scenario test | Annually | Data loss confirmed < 5 minutes (RPO validation) |

### 5.6 Infrastructure Region Requirements

| Requirement | Specification |
|---|---|
| Primary region | us-east-1 (N. Virginia) — active-active across 3 AZs |
| DR region | us-west-2 (Oregon) — active-passive; Aurora Global Database |
| Database replication mode | Aurora Global Database with automatic replication to DR region |
| S3 replication | Cross-region replication enabled for evidence buckets |
| Kafka | Single-region MSK; no cross-region Kafka replication (events can be replayed from outbox if needed) |

---

*This document contains NFRs only. Monitoring configuration, alert queries, runbook procedures, and deployment specifications are maintained separately.*