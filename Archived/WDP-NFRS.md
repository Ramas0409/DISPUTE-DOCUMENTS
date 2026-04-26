# WDP-NFRS.md
**Worldpay Dispute Platform — Non-Functional Requirements**
*Version: 2.0 | Rebuilt: April 2026*
*Source: v1.0 document (corrected) + April 2026 component survey*

---

## How to Read This Document

NFRs are organised into five confirmed sections (Performance, Availability and
Resilience, Security and Compliance, Scalability, Operational Constraints) followed
by a new sixth section: the Platform Risk Register.

**Sections 1–5** carry forward NFR targets from v1.0, with three categories of
correction applied:
- Removed entries that referenced Resilience4j circuit breakers (DEC-014 VOID)
  or BRE step checkpointing (DEC-011 VOID)
- Corrected the Kafka delivery guarantee claim (at-most-once, not at-least-once)
- Added exception notes where confirmed component behaviour conflicts with a stated NFR

**Section 6 — Platform Risk Register** is new. It documents confirmed gaps and
risks identified during the April 2026 component survey (all 40 DRAFT component
files). Each risk references the relevant ADR in WDP-DECISIONS.md v2.0.

**NFR targets:** Product team has not yet finalised NFR targets for the gaps
documented in Section 6. Targets will be added to this document once confirmed.
Do not apply NFR targets from this document to architecture decisions without
first confirming they are still current with the solution architect.

**Flags used:**
- ⚠️ OUTDATED — present in source docs but superseded or inconsistent with
  later evidence; retained for review
- ⚠️ VERIFY — target appears in planning or prior docs only and has not been
  validated against current production component configuration
- ⚠️ EXCEPTION — a confirmed component behaviour conflicts with this NFR;
  see referenced decision record

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
| Full evidence flow end-to-end | < 3 minutes | Target | — | — |
| Full evidence flow end-to-end | < 5 minutes | P95 | — | — |
| Full evidence flow end-to-end | < 10 minutes | P99 | — | — |

⚠️ OUTDATED — An earlier document states ACK generation target of < 60 seconds
at P99. The production SLO document does not repeat this figure and instead
defines ACK generation success rate (> 95%) as the primary SLO. Verify which
metric governs ACK SLA in production.

---

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

⚠️ OUTDATED — An earlier document references a File Processor capable of
scaling to 350 pods. Current Stage 1 production auto-scaling caps the File
Processor at 10 instances to prevent S3 throttling. The 350-pod figure is
obsolete.

---

### 1.3 API Response Latency (Stage 1 and Stage 2)

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

Note: v1.0 contained a row "BRE validation per step — < step-specific timeout."
This referenced BRE step checkpointing (DEC-011), which is void — the named
steps do not exist in the BusinessRulesProcessor codebase. This row has been
removed.

---

### 1.4 Stage 3 Analytics Performance ⚠️ VERIFY

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

---

### 2.2 Recovery Time and Recovery Point Objectives

| Failure Scenario | RTO | RPO | Recovery Mechanism |
|---|---|---|---|
| Pod failure | 30 seconds | 0 | Kubernetes self-healing |
| Node failure | 2 minutes | 0 | Auto-scaling, pod rescheduling |
| Availability zone failure | 5 minutes | 0 | Multi-AZ deployment |
| Database corruption | 30 minutes | 5 minutes | Aurora PITR from automated backups |
| Region failure (full DR) | 1 hour | 5 minutes | Aurora Global Database failover to us-west-2 |
| Complete data centre loss | 4 hours | 15 minutes | Cross-region DR site |

⚠️ OUTDATED — An earlier document states RTO = 4 hours and RPO = 5 minutes as
general targets. The infrastructure document provides the granular breakdown
above, superseding the general figure. The 4-hour / 5-minute combination
applies only to the worst-case full-region-failure scenario.

⚠️ OUTDATED — Evidence file processing design states RTO = 1 hour, RPO = 5
minutes for that component specifically. This conflicts with the full-region
RTO of 1 hour. The 1-hour evidence component figure appears to refer to
application recovery, not full DR. Verify the governing DR SLA.

---

### 2.3 Resilience Behaviour ⚠️ Corrected from v1.0

The v1.0 document described circuit breaker thresholds for Document Management,
card network integrations, and the Encryption API. **All circuit breaker entries
have been removed.** DEC-014 (Resilience4j circuit breakers) is void — Resilience4j
is confirmed absent from all 40 WDP components. See WDP-DECISIONS.md DEC-014.

The v1.0 delivery guarantee claim "At-least-once; offset committed only after
full processing" has been corrected. The platform uses at-most-once delivery —
see DEC-005.

| Requirement | Confirmed Specification |
|---|---|
| Kafka consumer delivery guarantee | **At-most-once.** Offset committed before processing begins (pre-commit). Events lost on pod crash have no automatic recovery. |
| Retry mechanism for external calls | Spring Retry (`@Retryable`) — present in a subset of components only. Typically 3 attempts with fixed delay. COMP-34 has no retry. |
| Outbox retry — transient failures | At least 2 retry attempts tracked via `wdp.outgoing_event_outbox` status progression before terminal ERROR (COMP-43 pattern). Not universally implemented. |
| Notification channel isolation | Per-channel outbox table; failure in one channel (e.g. CORE_EVENTS) does not affect other channels (e.g. BEN, Signifyd). |
| ACK generation success rate SLO | > 95% rolling 1 hour; alert at < 90% (CRITICAL) |
| Kafka broker durability | 3 brokers, 3 AZs; acks=all; min in-sync replicas = 2; idempotent producer enabled |

**What no longer applies (removed from v1.0):**

The following requirements from v1.0 Section 2.3 are removed because they
reference components that do not exist:
- Document Management API circuit breaker thresholds
- Network integration circuit breaker thresholds
- Encryption API circuit breaker scope
- DEK cache window on KMS outage (6-hour figure — unconfirmed from any
  component file; verify with EncryptionService team before reinstating)
- Merchant isolation via per-merchant circuit breakers

**NFR gap — resilience targets:** No confirmed NFR targets exist for acceptable
event loss rate, maximum acceptable hung-thread duration, or external dependency
timeout. Product team to define. See RISK-001, RISK-002, RISK-003 in Section 6.

---

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

---

### 3.2 PAN Handling Constraints

| Requirement | Specification |
|---|---|
| Plaintext PAN scope | Only EncryptionService (COMP-35) may handle plaintext PAN; no other component may store or process it |
| PAN at rest | Must not be stored in plaintext anywhere in the system ⚠️ EXCEPTION — see below |
| PAN in transit | Must not travel in plaintext across any network boundary or through Kafka |
| PAN in logs | Must never appear in application logs, metrics, or traces |
| PAN in ACK files | ACK files must use merchant-provided chargeback identifier, not PAN |
| Encryption algorithm | AES-256-GCM |
| Tokenisation algorithm | HMAC-SHA256 (for HPAN — deterministic, non-reversible) |
| Key storage | AWS KMS with FIPS 140-2 Level 3 validated HSMs; keys never leave HSM |

⚠️ EXCEPTION — DEC-019: CaseManagementService (COMP-23) standard case creation
(`POST /{platform}/case`) writes clear PAN to `nap.case.I_ACCI_CDH` and
`wdp.CASE.I_ACCT_CDH` before encryption occurs. PAN encryption only takes place
during the transaction enrichment flow, which is a secondary path restricted to
PIN and CORE platforms. This is a PCI-DSS compliance gap formally recorded as
an accepted risk in WDP-DECISIONS.md DEC-019. Database access controls are the
interim mitigation. Remediation path: move PAN encryption into the standard
case creation transaction.

---

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

---

### 3.4 Access Control

| Requirement | Specification |
|---|---|
| Service-to-service authentication | OAuth 2.0 client credentials (JWT) plus mutual TLS (mTLS) |
| PAN decrypt scope — full | ⚠️ VERIFY — v1.0 states "Chargeback Worker only." Current confirmed: COMP-43 CoreNotificationConsumer decrypts HPAN to clear PAN for DB2 new case inserts. COMP-34 MerchantTransactionService decrypts transiently via COMP-35 for settlement display. Confirm full scope. |
| PAN decrypt scope — masked | ⚠️ VERIFY — v1.0 states "Evidence Worker (first 6 + last 4 digits only)." Map to current component |
| PAN encrypt scope | ⚠️ VERIFY — v1.0 states "File Processor and API Processor only." Confirmed: COMP-07 and COMP-08 encrypt PAN on ingest. COMP-23 intended but DEC-019 exception active. |
| User authentication | JWT via shared IDP; OAuth 2.0 |
| JWT validation | Public key with `kid` claim (Phase 1); JWKS URL (Phase 2 — planned) |
| Merchant data isolation | Row-level isolation by merchant_id enforced at API level |
| Merchant API — additional auth layer | API Key required in addition to OAuth JWT for external merchants |
| Merchant API — rate limit | 1,000 requests per hour per merchant |
| Operations user roles | Tier 1 Agent, Tier 2 Agent, Team Lead, Manager, Admin — with distinct action permissions |
| Queue access | Users may only access disputes in queues assigned to their role |
| Operation-level RBAC | ⚠️ EXCEPTION — DEC-018: RBAC enforcement not active in CaseActionService (COMP-24). See Section 6, RISK-005. |

---

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
| GDPR right to erasure | PAN deleted from pan_store within 30 days of request; HPAN nulled on crypto_audit record; audit record itself retained under legal obligation |
| GDPR right to access | Available via admin API query within 30 days |
| CCPA deletion | Available via admin API within 90 days of request |

---

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
| File Processor (COMP-11) | 3 | 10 | CPU > 70% or memory > 80% |
| Publisher Scheduler (COMP-12) | 2 | 2 (fixed) | Leader election; no further horizontal scaling |
| Chargeback Worker ⚠️ VERIFY component identity | — | — | Kafka consumer lag > 1,000 per pod |
| Evidence Worker ⚠️ VERIFY — likely COMP-15 EvidenceConsumer | 5 | 20 | Kafka consumer lag > 1,000; max capped by Kafka partition count |
| EKS cluster nodes | 10 | 50 | CPU/memory utilisation; auto-scaling |

**Publisher Scheduler (COMP-12) constraint:** Uses a database-level advisory
lock for single-writer guarantees. Only one instance is active as writer at any
time; the second instance is a hot standby for failover only. This component
cannot be horizontally scaled for throughput.

**Additional confirmed scaling constraints from April 2026 survey:**

| Component | Constraint | Reason |
|---|---|---|
| VisaDisputeBatch (COMP-07) | Replica must equal exactly 1 | Parallel replicas poll the same external queue — see DEC-023 |
| FirstChargebackBatch (COMP-08) | Replica must equal exactly 1 | Same reason — DEC-023 |
| BusinessRulesProcessor (COMP-16) | Kafka consumer concurrency = 1 per replica | Single-threaded consumer; hung thread stalls all processing for that instance |

---

### 4.2 Kafka Partition Constraints

| Constraint | Value | Implication |
|---|---|---|
| wdp.file.outbox.events — partition count | 6 | Maximum effective parallel consumer instances = 6 without repartitioning |
| Partition key — stated standard | merchantId | All events for a merchant go to the same partition |
| Partition key — confirmed deviation | caseNumber used by all business-rules publishers and COMP-16 | See DEC-003 deviation map in WDP-DECISIONS.md |
| Top-5 merchant volume share | ~40% of total | These merchants create hot partitions; monitor lag on high-volume partitions |
| Partition count change | Requires topic recreation or manual rebalancing | Treat partition count as a semi-permanent decision |

⚠️ VERIFY — An archived document (v3.0) refers to 12 partitions for the
evidence topic and 50 partitions for the file outbox topic. The infrastructure
document specifies 6 partitions for wdp.file.outbox.events. Confirm actual
production partition counts before planning scaling.

---

### 4.3 Database Scalability

| Requirement | Specification |
|---|---|
| Aurora read replicas — minimum | 2 (production) |
| Aurora read replicas — auto-scale maximum | 5 |
| Auto-scale trigger | CPU > 70% or connection count > 700 |
| Scale-in cooldown | 5 minutes |
| Scale-out cooldown | 60 seconds |
| Connection pool maximum per service | 20 connections (HikariCP) |

---

### 4.4 MSK Storage Constraint — Permanent Floor

MSK provisioned storage is one-directional: once storage is scaled up, it
cannot be reduced. Every storage scaling event sets a new permanent floor.
This constraint applies across all environments.

**Current provisioned storage:** 500 GB per broker (3 brokers = 1,500 GB total).

All capacity planning decisions must treat any storage increase as an
irreversible commitment. The recommended utilisation ceiling before scaling
is 60% to maintain headroom.

---

### 4.5 Stage 3 Scalability ⚠️ VERIFY

| Requirement | Specification |
|---|---|
| Analytics concurrent user target | 1,000+ simultaneous users |
| ETL daily volume target | 1,000,000+ cases per day |
| S3 data lake growth | ~100 TB projected over time (rough estimate) |
| Redshift cluster initial sizing | Undetermined — open question as of November 2025 |
| ML inference scaling | Independent containerised microservice, GPU-capable instances |

---

## 5. Operational Constraints

### 5.1 Deployment Windows

| Environment | Deployment Frequency | Approval | Rollback SLA |
|---|---|---|---|
| Development | On every commit (automated) | Not required | 5 minutes |
| Staging | Daily (automated) | Not required | 10 minutes |
| Production | Weekly (manual trigger) | Architect or tech lead approval required | 15 minutes |

**Standard production window:** Tuesday and Thursday, 10:00 AM – 2:00 PM ET.

**Emergency deployments:** Permitted 24/7 with on-call engineer approval.

**Blackout period:** End of month is restricted due to chargeback volume spikes.
No production deployments during this window except P0 emergency patches.

---

### 5.2 Incident Response Constraints

| Requirement | Target |
|---|---|
| On-call alert acknowledgement | Within 5 minutes of PagerDuty page |
| CRITICAL alerts | Page on-call engineer immediately |
| WARNING alerts | Deliver to Slack channel; no immediate page |
| Post-incident review | Required for all P0 and P1 incidents |
| Blast radius determination | Required as first step of triage — single merchant vs. platform-wide |

---

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

---

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

---

### 5.5 Backup and Verification

| Requirement | Frequency | Pass Criteria |
|---|---|---|
| Aurora database PITR test | Weekly | Restore completed within 30 minutes; data integrity verified |
| Application config backup verification | Weekly | ConfigMaps and secrets restored successfully |
| Full DR drill (failover to us-west-2) | Quarterly | Full failover completed within 1 hour |
| Annual data loss scenario test | Annually | Data loss confirmed < 5 minutes (RPO validation) |

---

### 5.6 Infrastructure Region Requirements

| Requirement | Specification |
|---|---|
| Primary region | us-east-1 (N. Virginia) — active-active across 3 AZs |
| DR region | us-west-2 (Oregon) — active-passive; Aurora Global Database |
| Database replication mode | Aurora Global Database with automatic replication to DR region |
| S3 replication | Cross-region replication enabled for evidence buckets |
| Kafka | Single-region MSK; no cross-region Kafka replication (events can be replayed from outbox if needed) |

---

## 6. Platform Risk Register (April 2026 Component Survey)

This section documents confirmed gaps and risks from the April 2026 survey of all
40 WDP component files. Each risk has an assigned severity, the affected components,
and either a reference to the formal ADR in WDP-DECISIONS.md or a flag indicating
that no ADR yet exists.

NFR targets for addressing these risks have not been set. Product team to define.

**Severity key:**
- 🔴 CRITICAL — data loss, compliance failure, or silent split-brain possible
- 🟠 HIGH — significant operational risk; partial data integrity or security gap
- 🟡 MEDIUM — operational friction or incomplete feature with known workaround
- 🟢 LOW — informational; minor inconsistency or dead code with no current impact

---

### 6.1 Risk Summary Table

| Risk ID | Severity | Risk | Affected Components | ADR / Reference |
|---|---|---|---|---|
| RISK-001 | 🔴 | No circuit breakers on any external dependency | All 40 components | DEC-014 VOID |
| RISK-002 | 🔴 | No RestTemplate timeouts on any REST call | All components with REST | Component files confirmed |
| RISK-003 | 🔴 | At-most-once Kafka delivery — events lost on pod crash | All Kafka consumers | DEC-005 |
| RISK-004 | 🔴 | Clear PAN persisted on standard case creation | COMP-23 | DEC-019 |
| RISK-005 | 🟠 | RBAC not enforced in CaseActionService | COMP-24 | DEC-018 |
| RISK-006 | 🟠 | No idempotency on case creation | COMP-23 | DEC-020 |
| RISK-007 | 🟠 | BRE inconsistent failure handling across rule action types | COMP-16 | Component file |
| RISK-008 | 🟠 | BRE error visibility via SNOTE only — no error table | COMP-16 | Component file |
| RISK-009 | 🟠 | Non-atomic cross-datasource write on NAP case creation | COMP-23 | Open question |
| RISK-010 | 🟠 | UAMS wrong transaction manager — NAP schema partial writes | COMP-02 | DEC-021 |
| RISK-011 | 🟠 | BRE split-brain on Kafka publish failure | COMP-16 | DEC-001 deviation |
| RISK-012 | 🟠 | validateOrgId commented out — COMP-03 org authorization absent | COMP-03 | DEC-018 (related) |
| RISK-013 | 🟡 | Polling batch replica constraint has no automated enforcement | COMP-07, COMP-08 | DEC-023 |
| RISK-014 | 🟡 | removeItemFromQueueDisabled has no automated state check | COMP-07, COMP-08 | DEC-022 |
| RISK-015 | 🟡 | bre_orchestration_outbox PUBLISHED orphan rows — no auto-redrive | COMP-12, COMP-18 | Component file |
| RISK-016 | 🟡 | NAPOutcomeProcessor notesLookup commented out | COMP-39 | Component file |
| RISK-017 | 🟡 | VisaResponseQuestionnaire additionalImagesList silently discarded | COMP-40 | Component file |
| RISK-018 | 🟡 | AcceptService/ContestService — no rollback on Kafka publish failure | COMP-19, COMP-20 | DEC-001 deviation |
| RISK-019 | 🟡 | MerchantTransactionService — no retry on any of 10 external dependencies | COMP-34 | Component file |
| RISK-020 | 🟢 | LATAM platform silently dropped in BusinessRulesProcessor | COMP-16 | Component file |
| RISK-021 | 🟢 | DisputeService Kafka producer wired but commented out | COMP-22 | Component file |
| RISK-022 | 🟢 | BRE source field routing not implemented | COMP-16 | Component file |
| RISK-023 | 🟢 | DisplayCodeService does not determine TIER1 eligibility | COMP-28 | Component file |

---

### 6.2 Critical Risk Details

---

**RISK-001: No circuit breakers on any external dependency**
**Severity:** 🔴 CRITICAL | **Reference:** DEC-014 VOID

Resilience4j is confirmed absent from all 40 WDP components. No circuit breaker,
no rate limiter, and no bulkhead is configured on any outbound dependency anywhere
in the platform. BusinessRulesProcessor (COMP-16) is the highest-risk instance:
it calls seven downstream REST services with no timeout and no circuit breaker;
consumer concurrency is 1 per replica, so a single hung thread stalls all
`business-rules` message processing for that instance indefinitely with no
automatic recovery. The same pattern repeats across all WDP components that make
outbound REST calls.

**NFR gap:** No timeout targets or acceptable hung-thread duration defined.

---

**RISK-002: No RestTemplate timeouts on any WDP component**
**Severity:** 🔴 CRITICAL | **Reference:** Confirmed across all component files

No WDP component configures explicit connection or read timeouts on `RestTemplate`.
All outbound REST calls rely on OS-level TCP timeouts, which are effectively
infinite under normal network conditions. A downstream service that accepts the
TCP connection but never responds will hold the calling thread indefinitely. On
components with concurrency = 1 (COMP-16) this stalls the entire consumer. On
multi-threaded components this exhausts the thread pool over time.

**NFR gap:** No timeout threshold targets defined for any external dependency.

---

**RISK-003: At-most-once Kafka delivery — events lost on pod crash**
**Severity:** 🔴 CRITICAL | **Reference:** DEC-005

All WDP Kafka consumers commit the Kafka offset before processing begins. A pod
crash after the commit and before processing completes permanently loses the
event. There is no automatic redelivery, no Kafka DLQ, and no compensating
mechanism that detects missed events. The database error tables (DEC-016) only
record events that reach the error-handling path — events lost at the offset
commit boundary leave no trace.

**NFR gap:** No acceptable event loss rate target defined.

---

**RISK-004: Clear PAN persisted on standard case creation**
**Severity:** 🔴 CRITICAL | **Reference:** DEC-019

CaseManagementService (COMP-23) standard case creation writes the card number
in clear text to `nap.case.I_ACCI_CDH` and `wdp.CASE.I_ACCT_CDH`. PAN
encryption only occurs during the transaction enrichment flow, which is
restricted to PIN and CORE platforms. This is a live PCI-DSS compliance gap —
any database query, log capture, or backup restore that accesses these columns
exposes readable card numbers.

**Accepted risk:** Formally recorded in DEC-019. Interim mitigation is database
access controls. Remediation: move PAN encryption into the standard case creation
transaction.

---

### 6.3 High Risk Details

---

**RISK-005: RBAC not enforced in CaseActionService**
**Severity:** 🟠 HIGH | **Reference:** DEC-018

`RestInvoker.authorizeUser()` exists in COMP-24 CaseActionService but is never
called. Any authenticated user with entity-level access to a case can execute
any case action regardless of their assigned role. Role-based operation
restrictions — preventing a read-only user from submitting a chargeback, or a
junior operator from approving a write-off — are not enforced at the service
level.

**Related gap:** `validateOrgId()` is also commented out in COMP-03 on
`GET /orgentity` (RISK-012 below).

---

**RISK-006: No idempotency on case creation**
**Severity:** 🟠 HIGH | **Reference:** DEC-020

COMP-23 CaseManagementService performs no duplicate detection on case creation.
Concurrent identical HTTP requests produce multiple case records with different
case numbers. Under the current at-most-once delivery model (DEC-005), Kafka
redelivery cannot produce duplicates. The risk is limited to concurrent HTTP
requests. If the delivery model changes to at-least-once, this becomes a near-
certain operational event.

---

**RISK-007: BRE inconsistent failure handling across rule action types**
**Severity:** 🟠 HIGH | **Reference:** COMP-16 file

In BusinessRulesProcessor (COMP-16), different rule action types handle failures
differently:

| Action type | On failure |
|---|---|
| Add action | Re-throws `BusinessRulesException` — message dropped |
| Questionnaire | Exception swallowed — processing continues silently |
| Contest | Re-thrown — message dropped |
| Accept | Propagates uncaught — unpredictable outcome |
| Case update | Swallowed |
| Issuer doc | Swallowed |
| Outgoing Kafka publish | Swallowed |

Outcome for a given message depends entirely on which action type the matched
rule triggers. Some failures produce silent inconsistency; others halt processing
entirely. There is no uniform failure contract.

---

**RISK-008: BRE error visibility via SNOTE only**
**Severity:** 🟠 HIGH | **Reference:** COMP-16 file

`ErrorLogService` in COMP-16 is commented out. Errors are written as SNOTE
notes via a REST call to NotesService (COMP-25). If the NotesService call also
fails, the error is silently lost with no trace in any database table, no log
alert, and no error outbox row. This makes BRE processing failures the least
visible failure mode on the platform.

---

**RISK-009: Non-atomic cross-datasource write on NAP case creation**
**Severity:** 🟠 HIGH | **Reference:** Open question — COMP-23

The NAP case creation path in COMP-23 writes to `wdp.dispute_event_change_log`
(wdp schema) within the same logical flow as the NAP case write (nap schema).
The two schemas are managed by different datasources and cannot be in the same
transaction. On failure between the two writes, one datasource commits while
the other does not. Whether this is an accepted risk or a gap requires an
architect decision (open in WDP-HANDOVER.md).

---

**RISK-010: UAMS wrong transaction manager**
**Severity:** 🟠 HIGH | **Reference:** DEC-021 (Known Defect)

`saveChildWithMerchant` in COMP-02 UAMS uses `@Primary wdpTransactionManager`
for NAP schema writes. On failure, the rollback targets the wrong datasource.
NAP schema writes are not rolled back, leaving the NAP user or entity ACL in a
partially written state. This is a confirmed implementation defect, not an
accepted design decision.

---

**RISK-011: BRE split-brain on Kafka publish failure**
**Severity:** 🟠 HIGH | **Reference:** DEC-001 deviation

BusinessRulesProcessor (COMP-16) updates case state via REST (step 18) and then
publishes an outgoing event to COMP-18 NotificationOrchestrator via Kafka (in a
`finally` block). If the Kafka publish fails or is swallowed, the case state is
updated but no downstream event is delivered. COMP-18 never receives the event;
the dispute lifecycle stalls silently. No error record is created for this split-
brain condition.

---

**RISK-012: validateOrgId commented out in COMP-03**
**Severity:** 🟠 HIGH | **Reference:** Related to DEC-018

`validateOrgId()` is commented out in CoreHierarchyAuthorizationService (COMP-03)
on `GET /orgentity`. The org-level authorization check that should gate this
endpoint is absent. This is a second RBAC-adjacent gap alongside DEC-018 (COMP-24)
and should be assessed as part of the same remediation effort.

---

### 6.4 Medium Risk Details

---

**RISK-013: Polling batch replica constraint has no automated enforcement**
**Severity:** 🟡 MEDIUM | **Reference:** DEC-023

COMP-07 and COMP-08 must run at replica = 1. If a replica count other than 1 is
applied — via a Helm values change, HPA configuration, or emergency scaling
action — duplicate case creation begins immediately with no alerting. There is
no Kubernetes admission webhook, no HPA configuration protecting this, and no
monitoring alert for replica count on these specific components.

---

**RISK-014: removeItemFromQueueDisabled has no automated state check**
**Severity:** 🟡 MEDIUM | **Reference:** DEC-022

Both COMP-07 and COMP-08 support a `removeItemFromQueueDisabled` flag that
suppresses all queue acknowledgements globally. If this flag is left `true`
after a maintenance window, every item will be reprocessed on every subsequent
batch run, creating duplicate case records. No alert fires when the flag is in
this state. Manual configuration review is the only detection mechanism.

---

**RISK-015: bre_orchestration_outbox PUBLISHED orphan rows have no auto-redrive**
**Severity:** 🟡 MEDIUM | **Reference:** COMP-12, COMP-18

Rows in `wdp.bre_orchestration_outbox` that reach PUBLISHED status but are never
consumed (due to relay failure or pod crash between publish and consumption) have
no automatic re-drive mechanism. Manual intervention is required to identify and
reset orphaned PUBLISHED rows. The shared table structure (COMP-12 Scheduler4 and
COMP-18 share the table via component discriminator) makes identifying orphans
dependent on correctly interpreting the discriminator value.

---

**RISK-016: NAPOutcomeProcessor notesLookup permanently disabled**
**Severity:** 🟡 MEDIUM | **Reference:** COMP-39

The `notesLookup()` step in COMP-39 is fully commented out. The `dataRecord`
field in every SRV118 payload sent to NAP-DPS is permanently null for all
events. If NAP-DPS requires this field for any dispute processing downstream,
the omission will manifest as silent incorrect processing in NAP-DPS rather than
a WDP error.

---

**RISK-017: VisaResponseQuestionnaire additionalImagesList silently discarded**
**Severity:** 🟡 MEDIUM | **Reference:** COMP-40

COMP-40 uploads only the primary questionnaire image from the Visa RTSI response
(`disputeAsImageResponseDescriptor`). The `additionalImagesList` is extracted
from the RTSI response but silently discarded — it is never uploaded to
DocumentManagementService. This is confirmed incomplete work in the COMP-40 file.
Any documents in the additional images list are permanently lost without error
or notification.

---

**RISK-018: AcceptService and ContestService — no rollback on Kafka publish failure**
**Severity:** 🟡 MEDIUM | **Reference:** DEC-001 deviation, COMP-19, COMP-20

Both COMP-19 AcceptService and COMP-20 ContestService publish to
`internal-integration-events` after committing case actions via REST to
CaseManagementService. If all Kafka retries are exhausted, the case action is
permanently committed but no `internal-integration-events` event is delivered.
COMP-39 NAPOutcomeProcessor and COMP-40 VisaResponseQuestionnaire never receive
the event. The dispute lifecycle stalls without any error surfaced to the
operations team.

---

**RISK-019: MerchantTransactionService has no retry on any external dependency**
**Severity:** 🟡 MEDIUM | **Reference:** COMP-34

COMP-34 MerchantTransactionService calls 10 external dependencies (MCM API, Visa
Pinned API, internal Transaction Management Service, CORE DB2, EncryptionService,
and others) using bare `RestTemplate` with no retry, no timeout, and no circuit
breaker. A single transient failure on any call propagates immediately as an error
to the calling component. This makes the transaction enrichment path fragile to
transient network events.

---

### 6.5 Low Risk Details

---

**RISK-020: LATAM platform silently dropped in BusinessRulesProcessor**
**Severity:** 🟢 LOW | **Reference:** COMP-16

Events with `platform = LATAM` in BusinessRulesProcessor produce no outgoing
event, no error record, and no log alert at WARNING or above. The LATAM constant
is defined in `ApplicationConstants` but has no routing branch in the rules
processor. Any LATAM dispute that reaches the `business-rules` topic is silently
discarded.

---

**RISK-021: DisputeService Kafka producer wired but commented out**
**Severity:** 🟢 LOW | **Reference:** COMP-22

DisputeService (COMP-22) is read-only — it owns no database state and performs
no writes. A Kafka producer to the `business-rules` topic is wired in the
codebase but all publish call sites are commented out. If accidentally re-enabled
without review, unexpected Kafka publishes would occur from a service not intended
to be a producer.

---

**RISK-022: BRE source field routing not implemented**
**Severity:** 🟢 LOW | **Reference:** COMP-16

The `source` field on inbound `BusinessRuleEvent` messages (values BRISUP,
BRMRUP, BRMCUP) is logged by COMP-16 but not used for routing. All three source
values follow identical processing paths. If future differentiation by event
source is required, code changes to COMP-16 are needed — the field is present
and propagated but the routing logic does not exist.

---

**RISK-023: DisplayCodeService does not determine TIER1 eligibility**
**Severity:** 🟢 LOW | **Reference:** COMP-28

DisplayCodeService (COMP-28) returns raw display code lists. It does not
determine TIER1 eligibility — that logic belongs to the calling service.
If a calling service incorrectly treats a code list response as an eligibility
decision, incorrect tier assignments result. This is an interface contract
documentation issue, not a defect in COMP-28 itself.

---

*This document covers non-functional requirements and confirmed platform risks.*
*NFR targets for Section 6 risks to be added by product team once finalised.*
*Implementation details, database schemas, and deployment specifications are*
*maintained in individual component files (WDP-COMP-[NN]-*.md).*
