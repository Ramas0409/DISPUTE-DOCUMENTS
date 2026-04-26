# WDP-NFRS.md
**Worldpay Dispute Platform — Non-Functional Requirements**
*Version: 2.1 | Reconciled: 2026-04-25*
*Source: v2.0 (April 2026 rebuild) + 2026-04-18/23/24/25 source-verification reconciliation*
*Reconciled entries: COMP-07, COMP-08, COMP-09, COMP-11, COMP-12, COMP-13, COMP-14, COMP-15, COMP-16, COMP-17, COMP-18, COMP-19, COMP-20, COMP-21, COMP-23, COMP-24, COMP-27, COMP-37, COMP-41, COMP-43*

---

## How to Read This Document

NFRs are organised into five confirmed sections (Performance, Availability and Resilience, Security and Compliance, Scalability, Operational Constraints) followed by Section 6: the Platform Risk Register.

**Sections 1–5** carry forward NFR targets from v1.0, with three categories of correction applied:
- Removed entries that referenced Resilience4j circuit breakers (DEC-014 VOID) or BRE step checkpointing (DEC-011 VOID)
- Corrected the Kafka delivery guarantee claim (at-most-once, not at-least-once)
- Added exception notes where confirmed component behaviour conflicts with a stated NFR

**Section 6 — Platform Risk Register.** The v2.0 register documented 23 confirmed risks (RISK-001 through RISK-023). The v2.1 reconciliation appends RISK-024 onwards from the 20-entry source-verification audit and withdraws RISK-009 (the underlying table did not exist in source).

**NFR targets:** Product team has not yet finalised NFR targets for the gaps documented in Section 6. Targets will be added once confirmed. Do not apply NFR targets from this document to architecture decisions without first confirming they are still current with the solution architect.

**Flags used:**
- ⚠️ OUTDATED — present in source docs but superseded or inconsistent with later evidence; retained for review
- ⚠️ VERIFY — target appears in planning or prior docs only and has not been validated against current production component configuration
- ⚠️ EXCEPTION — a confirmed component behaviour conflicts with this NFR; see referenced decision record
- ⚠️ WITHDRAWN — risk previously documented but invalidated by source verification

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

⚠️ OUTDATED — An earlier document states ACK generation target of < 60 seconds at P99. The production SLO document does not repeat this figure and instead defines ACK generation success rate (> 95%) as the primary SLO. Verify which metric governs ACK SLA in production.

⚠️ NEW (2026-04-18) — File job COMPLETED transition ownership undocumented (RISK-088). COMP-11 never writes COMPLETED; COMP-13 polls for `status IN (COMPLETED, ERROR)`; COMP-12 is the candidate writer per WDP-DB.md but the contract is undocumented in all three repos. ACK-generation SLO measurement is uncertain pending contract confirmation.

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

⚠️ OUTDATED — An earlier document references a File Processor capable of scaling to 350 pods. Current Stage 1 production auto-scaling caps the File Processor at 10 instances to prevent S3 throttling. The 350-pod figure is obsolete.

⚠️ EXCEPTION (2026-04-25) — COMP-21 ChargebackService throughput ceiling is far below replica-count expectations. The `asyncExecutor` is core=1, max=1, queue=5; the parallel ACL+lookup pattern provides no real parallelism. Seventh concurrent action request hits `RejectedExecutionException`. See RISK-026.

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

Note: v1.0 contained a row "BRE validation per step — < step-specific timeout." This referenced BRE step checkpointing (DEC-011), which is void — the named steps do not exist in the BusinessRulesProcessor codebase. This row has been removed.

⚠️ EXCEPTION (2026-04-25) — COMP-21 ChargebackService is the externally-visible WDP entry point for merchants. Its `contest` WDP-path performs ~10 sequential round-trips (5 business calls each preceded by a fresh IDP token GET, no token cache, no connection pool) on a bare `RestTemplate`. This sets a hard latency floor on the platform's externally-visible API. See RISK-025.

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

⚠️ EXCEPTION (2026-04-18/25) — Pod availability for several long-running components is not measurable in the standard sense because they have no Kubernetes liveness, readiness, or startup probes. Hung pods are not evicted by kubelet. Affected: COMP-07, COMP-08, COMP-09, COMP-11, COMP-12, COMP-14, COMP-17, COMP-41, COMP-43. See RISK-024.

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

⚠️ OUTDATED — An earlier document states RTO = 4 hours and RPO = 5 minutes as general targets. The infrastructure document provides the granular breakdown above, superseding the general figure.

⚠️ OUTDATED — Evidence file processing design states RTO = 1 hour, RPO = 5 minutes for that component specifically. This conflicts with the full-region RTO of 1 hour. The 1-hour evidence component figure appears to refer to application recovery, not full DR. Verify the governing DR SLA.

---

### 2.3 Resilience Behaviour ⚠️ Corrected from v1.0

The v1.0 document described circuit breaker thresholds for Document Management, card network integrations, and the Encryption API. **All circuit breaker entries have been removed.** DEC-014 is void — Resilience4j confirmed absent from all 40 WDP components.

The v1.0 delivery guarantee claim "At-least-once; offset committed only after full processing" has been corrected. The platform uses at-most-once delivery — see DEC-005.

| Requirement | Confirmed Specification |
|---|---|
| Kafka consumer delivery guarantee | **At-most-once** on Kafka consumers (COMP-14/15/16/17/18/39/41/42/43). **At-least-once with duplicate-possible window** on COMP-12 outbox→Kafka relays (mark-and-send within `@Transactional`; broker ACK precedes TX commit — *corrected 2026-04-18 from previously documented at-most-once*). Consumer-side `idempotency-key` dedup is the contracted mitigation. |
| Retry mechanism for external calls | Spring Retry (`@Retryable`) — present in a subset of components only. Typically 3 attempts with fixed delay. COMP-34 has no retry. ⚠️ **(2026-04-25)** COMP-41 imports `@Retryable`/`@Backoff` but never applies them — class names containing "Retry" describe custom try/catch, not framework. |
| Outbox retry — transient failures | At least 2 retry attempts tracked via `wdp.outgoing_event_outbox` status progression before terminal ERROR (COMP-43 pattern). Not universally implemented. ⚠️ **(2026-04-25)** Scheduler3 reads only FAILED and PENDING_DEFERRED rows — PUBLISHED orphans have no auto-redrive. See RISK-015 / RISK-040. |
| Notification channel isolation | Per-channel outbox table; failure in one channel (e.g. CORE_EVENTS) does not affect other channels (e.g. BEN, Signifyd). |
| ACK generation success rate SLO | > 95% rolling 1 hour; alert at < 90% (CRITICAL) — ⚠️ measurement reliability depends on COMPLETED-transition contract (see Section 1.1) |
| Kafka broker durability | 3 brokers, 3 AZs; acks=all; min in-sync replicas = 2; idempotent producer enabled |
| Bad-payload handling | ⚠️ **(2026-04-18+)** Empty `CommonErrorHandler{}` registered platform-wide on multiple consumers — silent drop on deserialisation failure with no DLT, no log, no counter. Distinct silent-loss class. See RISK-025. |

**What no longer applies (removed from v1.0):**
- Document Management API circuit breaker thresholds
- Network integration circuit breaker thresholds
- Encryption API circuit breaker scope
- DEK cache window on KMS outage (6-hour figure — unconfirmed; verify before reinstating)
- Merchant isolation via per-merchant circuit breakers

**NFR gap — resilience targets:** No confirmed NFR targets exist for acceptable event loss rate, maximum acceptable hung-thread duration, or external dependency timeout. Product team to define. See RISK-001, RISK-002, RISK-003 in Section 6.

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

⚠️ EXCEPTION — DEC-019 (PostgreSQL): CaseManagementService (COMP-23) standard case creation (`POST /{platform}/case`) writes clear PAN to `nap.case.I_ACCT_CDH` *(corrected 2026-04-23 from typo `I_ACCI_CDH` in v2.0)* and `wdp.CASE.I_ACCT_CDH` before encryption occurs. PAN encryption only takes place during the transaction enrichment flow, which is a secondary path restricted to PIN and CORE platforms. Database access controls are the interim mitigation. See RISK-004.

⚠️ EXCEPTION — DEC-019 (DB2): **(2026-04-25)** CoreNotificationConsumer (COMP-43) writes clear PAN to `BC.TBC_DM_CASE.I_ACCT_CDH` *(corrected from `I_ACCT_CDR`)* on Step 7 CREATE + actionSeq=01 path after Encryption Service decrypt at the Step 6 PAN gate. Whether this is intentional and approved is owed by the CORE platform team. See RISK-026.

⚠️ EXCEPTION — In-flight PAN surface: **(2026-04-25)** COMP-21 ChargebackService surfaces full `cardNumber` in two response model classes (`SearchCaseList`, `Transaction`) and `cardNumberLast4` in eight others. Populated from downstream `case-search-service` responses; no masking transformation in COMP-21 before serialisation. Whether clear PAN actually flows is downstream-dependent. See RISK-051.

⚠️ EXCEPTION — DEC-004 edge case: **(2026-04-18)** COMP-11 `NetworkFileSupport` encrypts only when `acctNum` matches `\d+`. The non-numeric branch passes the raw value into `chbk_outbox_row.payload.account_number`. Whether production DNWK files can contain non-numeric PAN is OQ-FileEdge. See RISK-073.

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
| PAN decrypt scope — full | Confirmed: COMP-43 CoreNotificationConsumer decrypts HPAN to clear PAN at Step 6 PAN gate (CREATE + actionSeq=01 path) for DB2 new case INSERT to `BC.TBC_DM_CASE.I_ACCT_CDH`. COMP-34 MerchantTransactionService decrypts transiently via COMP-35 for settlement display. |
| PAN decrypt scope — masked | ⚠️ VERIFY — v1.0 states "Evidence Worker (first 6 + last 4 digits only)." Map to current component |
| PAN encrypt scope | Confirmed: COMP-07 and COMP-08 encrypt PAN on ingest (per `\d+` gate in `NetworkFileSupport` — non-numeric edge case bypasses encryption). COMP-23 intended but DEC-019 exception active. |
| User authentication | JWT via shared IDP; OAuth 2.0 |
| JWT validation | Public key with `kid` claim (Phase 1); JWKS URL (Phase 2 — planned) |
| Merchant data isolation | Row-level isolation by merchant_id enforced at API level |
| Merchant API — additional auth layer | API Key required in addition to OAuth JWT for external merchants |
| Merchant API — rate limit | 1,000 requests per hour per merchant |
| Operations user roles | Tier 1 Agent, Tier 2 Agent, Team Lead, Manager, Admin — with distinct action permissions |
| Queue access | Users may only access disputes in queues assigned to their role |
| Operation-level RBAC | ⚠️ EXCEPTION — DEC-018: RBAC enforcement not active in CaseActionService (COMP-24). See RISK-005. |
| Internal vs external authorization | ⚠️ EXCEPTION (2026-04-24) — COMP-27 CaseSearchService `POST /lft` derives internal-vs-external authorization from a request-body field (`LftSearchParams.isInternal`), NOT from the JWT. See RISK-042. |
| Org-scope authorization on CHAS | ⚠️ EXCEPTION — `validateOrgId()` commented out in COMP-03 on `GET /orgentity`. See RISK-012. |
| External entity-scope authorization | ⚠️ EXCEPTION (2026-04-24) — External VAP and LATAM callers in COMP-27 are not routed through any entity-scope authorization service. Only NAP (→ UAMS) and PIN/CORE (→ CHAS) are. Intent unconfirmed. |
| Internal PIN regular role | **(2026-04-24)** `WDP_PIN_REGULAR` role exists in `AuthorizationList` alongside `WDP_NAP_REGULAR`. Filter behaviour: internal NAP/PIN regular users filtered to queue assignment; internal non-regular users receive unfiltered results. |
| Partner identity routing | **(2026-04-25)** COMP-21 ChargebackService identifies partners via `entitlement_params` consumer name: `SIGNIFYD` and `JUSTTAI` (double-T). ACL chain-ID validation is skipped for partners. |

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

⚠️ EXCEPTION (2026-04-25) — Logstash audit-pipeline reliability: COMP-21 production secret `logstash_server_host_port` is empty. Only stdout logs reach aggregation today. See RISK-050.

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
| Chargeback Worker ⚠️ VERIFY | — | — | Kafka consumer lag > 1,000 per pod |
| Evidence Worker (COMP-15 EvidenceConsumer) | 5 | 20 | Kafka consumer lag > 1,000; max capped by Kafka partition count |
| EKS cluster nodes | 10 | 50 | CPU/memory utilisation; auto-scaling |

**Publisher Scheduler (COMP-12) constraint:** ⚠️ EXCEPTION (2026-04-18) — Source verification reveals **no `@SchedulerLock`, no advisory lock, no `SELECT FOR UPDATE`, no `synchronized` guard**. Replicas > 1 is a 🔴 unmitigated concurrency race — duplicate Kafka publishes guaranteed. Replica count is XLD-templated and not visible in source. See RISK-038.

**Additional confirmed scaling constraints from April 2026 survey:**

| Component | Constraint | Reason |
|---|---|---|
| VisaDisputeBatch (COMP-07) | Replica must equal exactly 1 | Parallel replicas poll the same external queue — see DEC-023 |
| FirstChargebackBatch (COMP-08) | Replica must equal exactly 1 | Same reason — DEC-023 |
| CaseFillingBatch (COMP-09) | Replica must equal exactly 1 | Same pattern as COMP-07/08 — DEC-023 |
| FileProcessor (COMP-11) | Per-pod `maxConcurrentMessages=1` | SQS-listener serialisation; cross-pod race on `(file_name, s3_key)` mitigated only by SQS message visibility |
| BusinessRulesProcessor (COMP-16) | Kafka consumer concurrency = 1 per replica | Single-threaded consumer; hung thread stalls all processing for that instance |
| CaseExpiryUpdateConsumer (COMP-17) | Concurrency = 1 per replica | No singleton guard; replicas > 1 rely entirely on Kafka consumer-group rebalance |
| ThirdPartyNotificationConsumer (COMP-41) | Concurrency = 1, no `setConcurrency()` | Same pattern; auto.offset.reset=latest skips backlog on cold start |
| BENConsumer (COMP-42) | Concurrency = 1 (default) | Same pattern |
| CoreNotificationConsumer (COMP-43) | Concurrency = 1 (default), `max.poll.records=500`, `max.poll.interval.ms=600000` | Bad-payload rebalance loop possible — see RISK-068 |
| ChargebackService (COMP-21) | Per-pod `asyncExecutor` core=1, max=1, queue=5 | 7th concurrent action request hits `RejectedExecutionException`. Throughput ceiling is far below replica-count expectations. See RISK-026. |

---

### 4.2 Kafka Partition Constraints

| Constraint | Value | Implication |
|---|---|---|
| wdp.file.outbox.events — partition count | 6 | Maximum effective parallel consumer instances = 6 without repartitioning |
| Partition key — stated standard | merchantId | All events for a merchant go to the same partition |
| Partition key — confirmed deviation | `caseNumber` used by all six business-rules publishers (COMP-12, 15, 23, 24, 25, 37); pass-through `KafkaHeaders.RECEIVED_KEY` on COMP-18 outbound topics; compound `networkCaseId+cardNetwork+platform` on COMP-12 → COMP-14 path | See DEC-003 deviation map in WDP-DECISIONS.md |
| Top-5 merchant volume share | ~40% of total | These merchants create hot partitions; monitor lag on high-volume partitions |
| Partition count change | Requires topic recreation or manual rebalancing | Treat partition count as a semi-permanent decision |

⚠️ VERIFY — An archived document (v3.0) refers to 12 partitions for the evidence topic and 50 partitions for the file outbox topic. The infrastructure document specifies 6 partitions for `wdp.file.outbox.events`. Confirm actual production partition counts before planning scaling.

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

MSK provisioned storage is one-directional: once storage is scaled up, it cannot be reduced. Every storage scaling event sets a new permanent floor. This constraint applies across all environments.

**Current provisioned storage:** 500 GB per broker (3 brokers = 1,500 GB total).

All capacity planning decisions must treat any storage increase as an irreversible commitment. The recommended utilisation ceiling before scaling is 60% to maintain headroom.

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

**Blackout period:** End of month is restricted due to chargeback volume spikes. No production deployments during this window except P0 emergency patches.

⚠️ EXCEPTION (2026-04-23) — Several production component images ship `spring-boot-devtools` (COMP-23 confirmed; COMP-24 likely). Dev-time class-path scanning may execute in production. See RISK-076.

---

### 5.2 Incident Response Constraints

| Requirement | Target |
|---|---|
| On-call alert acknowledgement | Within 5 minutes of PagerDuty page |
| CRITICAL alerts | Page on-call engineer immediately |
| WARNING alerts | Deliver to Slack channel; no immediate page |
| Post-incident review | Required for all P0 and P1 incidents |
| Blast radius determination | Required as first step of triage — single merchant vs. platform-wide |

⚠️ EXCEPTION (2026-04-18+) — Incident correlation gaps:
- COMP-12 Scheduler3/Scheduler4 paths do not persist Kafka metadata (`kafka_offset`, `kafka_partition`, `kafka_topic`) to outbox rows. Correlation between Kafka logs and outbox rows on those paths requires log-side join on `idempotencyId`.
- COMP-43 outbox row entity carries no Kafka coordinate columns — same incident-correlation gap.
- COMP-17 `v-correlation-id` is not propagated on the IDP token call (only on case-search call). Correlation chain breaks at IDP boundary. See RISK-079.
- COMP-43 has no MDC enrichment, no custom Micrometer metrics. No counters for SKIPPED / ERROR / FAILED / SUCCESS outcomes.

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

⚠️ NOTE (2026-04-23) — COMP-08 SKIPPED-marker accumulation: one row per re-polled known chargeback per scheduler run. Whether Scheduler2 archives SKIPPED rows is OQ-COMP08-Archive. See RISK-061.

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

⚠️ EXCEPTION (2026-04-25) — Several components have no `management:` block in YAML profiles, exposing Spring Boot Actuator defaults only (`health` and `info`); Prometheus metrics not exposed. Affected: COMP-43, COMP-17, COMP-18 (default exposure only). Custom Micrometer meters are absent across most components.

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

⚠️ NOTE (2026-04-18) — COMP-11 `S3ClientConfiguration` hardcodes AWS region `us-east-2` (not `us-east-1`). Reconcile against infrastructure documentation.

---

## 6. Platform Risk Register

This section documents confirmed gaps and risks. RISK-001 through RISK-023 were identified during the April 2026 component survey of all 40 component files. RISK-024 onwards were added during the 2026-04-18/23/24/25 source-verification reconciliation pass against 20 components.

NFR targets for addressing these risks have not been set. Product team to define.

**Severity key:**
- 🔴 CRITICAL — data loss, compliance failure, or silent split-brain possible
- 🟠 HIGH — significant operational risk; partial data integrity or security gap
- 🟡 MEDIUM — operational friction or incomplete feature with known workaround
- 🟢 LOW — informational; minor inconsistency or dead code with no current impact

---

### 6.1 Risk Summary Table

#### Phase 1 — April 2026 component survey (RISK-001 to RISK-023)

| Risk ID | Severity | Risk | Affected Components | ADR / Reference |
|---|---|---|---|---|
| RISK-001 | 🔴 | No circuit breakers on any external dependency | All 40 components | DEC-014 VOID |
| RISK-002 | 🔴 | No RestTemplate timeouts on any REST call | All components with REST | Component files confirmed |
| RISK-003 | 🔴 | At-most-once Kafka delivery — events lost on pod crash | All Kafka consumers | DEC-005 |
| RISK-004 | 🔴 | Clear PAN persisted on standard case creation (PostgreSQL) | COMP-23 | DEC-019 |
| RISK-005 | 🟠 | RBAC not enforced in CaseActionService | COMP-24 | DEC-018 |
| RISK-006 | 🟠 | No idempotency on case creation | COMP-23 | DEC-020 |
| RISK-007 | 🟠 | BRE inconsistent failure handling across rule action types | COMP-16 | Component file |
| RISK-008 | 🟠 | BRE error visibility via SNOTE only — no error table | COMP-16 | Component file |
| RISK-009 | ⚠️ WITHDRAWN (2026-04-23) | Was: non-atomic cross-datasource write on NAP case creation. **Withdrawn:** `wdp.dispute_event_change_log` table does not exist in source. | — | — |
| RISK-010 | 🟠 | UAMS wrong transaction manager — NAP schema partial writes | COMP-02 | DEC-021 |
| RISK-011 | 🟠 | BRE split-brain on Kafka publish failure | COMP-16 | DEC-001 deviation |
| RISK-012 | 🟠 | validateOrgId commented out — COMP-03 org authorization absent | COMP-03 | DEC-018 (related) |
| RISK-013 | 🟡 | Polling batch replica constraint has no automated enforcement | COMP-07, COMP-08, COMP-09 | DEC-023 |
| RISK-014 | 🟡 | removeItemFromQueueDisabled has no automated state check | COMP-07, COMP-08 | DEC-022 |
| RISK-015 | 🟡 | bre_orchestration_outbox PUBLISHED orphan rows — no auto-redrive | COMP-12, COMP-18 | Component file (extended by RISK-040) |
| RISK-016 | 🟡 | NAPOutcomeProcessor notesLookup commented out | COMP-39 | Component file |
| RISK-017 | 🟡 | VisaResponseQuestionnaire additionalImagesList silently discarded | COMP-40 | Component file |
| RISK-018 | 🟡 | AcceptService/ContestService — no rollback on Kafka publish failure | COMP-19, COMP-20 | DEC-001 deviation |
| RISK-019 | 🟡 | MerchantTransactionService — no retry on any of 10 external dependencies | COMP-34 | Component file |
| RISK-020 | 🟢 | LATAM platform silently dropped in BusinessRulesProcessor | COMP-16 | Component file |
| RISK-021 | 🟢 | DisputeService Kafka producer wired but commented out | COMP-22 | Component file |
| RISK-022 | 🟢 | BRE source field routing not implemented | COMP-16 | Component file |
| RISK-023 | 🟢 | DisplayCodeService does not determine TIER1 eligibility | COMP-28 | Component file |

#### Phase 2 — 2026-04-18/23/24/25 source-verification reconciliation (RISK-024 onwards)

| Risk ID | Severity | Risk | Affected Components | ADR / Reference |
|---|---|---|---|---|
| RISK-024 | 🔴 | No K8s liveness/readiness/startup probes — kubelet cannot evict hung pods | COMP-07, 08, 09, 11, 12, 14, 17, 41, 43 | Component files |
| RISK-025 | 🔴 | Empty `CommonErrorHandler{}` silent swallow — distinct silent-loss class platform-wide | COMP-14, 15, 16, 17, 18, 39, 41, 42, 43 | Component files |
| RISK-026 | 🔴 | COMP-21 platform external API latency floor — no IDP token cache + asyncExecutor core=1 | COMP-21 | Component file |
| RISK-027 | 🔴 | COMP-21 LATAM silent fall-through — HTTP 200 empty body, no log/metric/error | COMP-21 | Component file |
| RISK-028 | 🔴 | COMP-19 NAP split-brain on MC CHI and AMEX/DISCOVER — `AcceptEvent` published without network notification | COMP-19 | ADR pending |
| RISK-029 | 🔴 | COMP-15 V3 silent-ATTACHED defect — MISCDOC/DRFTDOC/RESPQDOC/ISSRQDOC mark `attachment_status=ATTACHED` with no upload | COMP-15 | Component file |
| RISK-030 | 🔴 | COMP-15 V3 PATCH ghost-upload — V3 Core has document, WDP DB unmarked | COMP-15 | Component file |
| RISK-031 | 🔴 | COMP-17 non-atomic outbox + `case_expiry` split — unrecoverable on every successful execution path | COMP-17 | Component file |
| RISK-032 | 🔴 | COMP-23 duplicate-NOTES-insert defect on US create path — two identical rows per create when notesRequest≠null | COMP-23 | Component file |
| RISK-033 | 🔴 | COMP-23 blind-merge on `NAP.DISPUTE_EVENT_CONSUMER_ERROR` — cross-component write into COMP-05-owned table with no owner check | COMP-23 | Component file |
| RISK-034 | 🔴 | COMP-37 5-step non-atomic write chain (S3 → DDB → desk → Kafka → action-indicator) — no compensation at any boundary | COMP-37 | DEC-001 deviation |
| RISK-035 | 🔴 | COMP-43 clear PAN persisted to `BC.TBC_DM_CASE.I_ACCT_CDH` (DB2) on CREATE+actionSeq=01 | COMP-43 | DEC-019 (DB2 sibling of RISK-004) |
| RISK-036 | 🔴 | COMP-43 silent-loss window between ACK and FAILED-write — PostgreSQL unavailability defeats fallback | COMP-43 | DEC-005 / DEC-001 |
| RISK-037 | 🔴 | COMP-43 REST PUT to WDP Case Actions Service inside DB2 `@Transactional` on CREATE — REST latency holds DB2 locks | COMP-43 | Component file |
| RISK-038 | 🔴 | COMP-12 replicas>1 unmitigated concurrency race — no `@SchedulerLock`, no advisory lock, no `SELECT FOR UPDATE`, no `synchronized` guard | COMP-12 | DEC-023 (extension) |
| RISK-039 | 🔴 | COMP-24 EP-9 three-transaction sequence with no compensation — RRSP action commits, Document Service POST, merchantDocIndicator update commit independently | COMP-24 | DEC-001 deviation |
| RISK-040 | 🔴 | COMP-41 three distinct PUBLISHED-orphan paths invisible to Scheduler3 — (a) post-ACK crash, (b) Signifyd "NO_DATA_FROM_SIGNIFYD" empty body, (c) final outbox UPDATE failure | COMP-41 | Extends RISK-015 |
| RISK-041 | 🔴 | COMP-27 concurrent-locker race on case lock UPDATE — no `@Transactional`, no `SELECT FOR UPDATE`, no optimistic version | COMP-27 | DEC-020 (PARTIAL) |
| RISK-042 | 🔴 | COMP-27 `/lft` authorization derives from request body (`isInternal`), not JWT | COMP-27 | Security gap |
| RISK-043 | 🔴 | COMP-27 queue-criterion `value` SQL concatenation — latent SQL-injection surface | COMP-27 | Component file |
| RISK-044 | 🟠 | COMP-19 compensation has no inner try/catch — secondary failure replaces original business exception, masks root cause (likely shared with COMP-20) | COMP-19, COMP-20 | Component file |
| RISK-045 | 🟠 | COMP-17 cross-action predecessor interference — predecessor query not scoped on `i_action_seq`; stuck row for action A blocks events for action B on same case | COMP-17 | Component file |
| RISK-046 | 🟠 | COMP-24 ActionEvent post-commit split-brain on EP 2/8/9 (when `napUpdateEvent=true`) — domain commit succeeds, ActionEvent publish lost on broker failure | COMP-24 | DEC-001 PARTIAL |
| RISK-047 | 🟠 | COMP-24 NAP/US asymmetry on EP 5 — `UKCaseActionDaoImpl.updateAction` does not process `chbkOutbox` on any branch; NAP silently ignores field US handles | COMP-24 | Component file |
| RISK-048 | 🟠 | COMP-24 open-action constraint enforced in-memory only — race window on concurrent POSTs for same caseNumber | COMP-24 | DEC-020 (PARTIAL) |
| RISK-049 | 🟠 | COMP-21 `IdpRestInvoker.setErrorHandler` mutates global `RestTemplate` per token call — concurrency hazard on shared bean | COMP-21 | Component file |
| RISK-050 | 🟠 | COMP-21 logstash appender effectively broken — production secret `logstash_server_host_port` is empty; only stdout logs reach aggregation | COMP-21 | Component file |
| RISK-051 | 🟠 | COMP-21 surfaces full `cardNumber` in two response models (`SearchCaseList`, `Transaction`) — DEC-004 in-flight surface gated on downstream | COMP-21 | DEC-004 (in-flight) |
| RISK-052 | 🟠 | COMP-08 writer-ACK hazard — mid-chunk JPA save failure does not prevent ACK PUT to MCM; ACK exceptions swallowed silently | COMP-08 | Component file |
| RISK-053 | 🟠 | COMP-09 writer swallows all save exceptions — chunk reports success even on DB save failure | COMP-09 | Component file |
| RISK-054 | 🟠 | COMP-11 DISCHYB and AMEXOPTB silent file loss — no bean for `DISCHYB_NETWORK` qualifier; no `FileAcroEnum` prefix entry for AMEXOPTB | COMP-11 | Component file |
| RISK-055 | 🟠 | COMP-41 silent `@Cacheable` no-op — `@EnableCaching` absent, no `CacheManager` bean. Every event hits upstream Display Code POST + Notification Rule GET | COMP-41 | Component file |
| RISK-056 | 🟠 | COMP-13 three latent runtime bugs surfaced 2026-04-18 (HIGH severity) including third `headObject` failure branch | COMP-13 | Component file |
| RISK-057 | 🟡 | COMP-43 empty `CommonErrorHandler` rebalance loop — bad payload causes NPE; no ACK, redelivers up to `max.poll.interval.ms`, then expelled. Persistent bad payloads can produce a rebalance loop | COMP-43 | Subset of RISK-025 |
| RISK-058 | 🟡 | COMP-43 UPDATE path runs `coreCaseRepository.save` outside any explicit `@Transactional` — Spring Data wraps in per-call short coreTx; CREATE-path symmetry missing | COMP-43 | Component file |
| RISK-059 | 🟡 | COMP-43 idempotency check SELECT and INSERT run in separate JPA short transactions — race window wider than v1.0 implied; no DB-level UNIQUE visible | COMP-43 | DEC-020 PARTIAL |
| RISK-060 | 🟡 | COMP-43 `actionSequence` comparison is `equalsIgnoreCase("01")` — no leading-zero normalisation. `"1"` skips PAN decryption AND treats case as subsequent occurrence | COMP-43 | Contract-edge brittleness |
| RISK-061 | 🟡 | COMP-08 SKIPPED-marker accumulation — one row per re-polled known chargeback per scheduler run; archive boundedness unconfirmed | COMP-08 | Component file |
| RISK-062 | 🟡 | COMP-08 update-path PENDING with `accountNumber=null` — `processUpdatedClaims` never sets `isAccountNumberRequired=true`. Suspected defect | COMP-08 | Component file |
| RISK-063 | 🟡 | COMP-09 skip paths write no row at all — null-validation, stage-determination, IDP token, encryption failures leave zero database trace | COMP-09 | Component file |
| RISK-064 | 🟡 | COMP-11 DCPO evidence loop is empty-catch orphan generator — parent CHARGEBACK_PROCESS persists at PENDING, evidence loop fails silently | COMP-11 | Component file |
| RISK-065 | 🟡 | COMP-11 DNWK per-record silent row drop — second save exception advances `rowCount`; resume logic skips the row | COMP-11 | Component file |
| RISK-066 | 🟡 | COMP-37 DynamoDB duplicate-check race — concurrent identical `(caseNumber, actionSequence, documentName)` both pass the non-atomic application-level check; later `putItem` silently overwrites earlier row including insert-audit fields | COMP-37 | DEC-020 PARTIAL |
| RISK-067 | 🟡 | COMP-23 `RequestCorrelation` ThreadLocal leak — latent cross-request contamination on pooled Tomcat worker threads | COMP-23 | Component file |
| RISK-068 | 🟡 | COMP-23 case-number sequence NPE — `getRandomDigits` returns null once sequence length + prefix + random alpha reaches 12; deterministic once sequence grows | COMP-23 | Component file |
| RISK-069 | 🟡 | COMP-23 `chbk_outbox_row` update path has no terminal-status guard — can overwrite SUCCESS/PROCESSED row on retry | COMP-23 | Component file |
| RISK-070 | 🟡 | COMP-21 `GET /cases/{id}` uses different source-system helper than action endpoints — same caseId may resolve to different source systems on read vs action | COMP-21 | Component file |
| RISK-071 | 🟡 | COMP-21 `/cases/{id}/changeowner` VAP path can fire two PUT calls; second wrapped in try/catch that swallows. Silent partial failure on second note | COMP-21 | Component file |
| RISK-072 | 🟡 | COMP-14 four `@Recover` methods on the enrichment chain return `null` silently — processing may reach SUCCESS on under-enriched case | COMP-14 | Component file |
| RISK-073 | 🟡 | COMP-11 DEC-004 non-numeric `acctNum` edge — `NetworkFileSupport` encrypts only when matches `\d+`; non-numeric branch passes raw into payload | COMP-11 | DEC-004 edge case |
| RISK-074 | 🟡 | COMP-11 `S3ServiceImpl` silently swallows get/move/put failures and returns null — not a throw path | COMP-11 | Component file |
| RISK-075 | 🟡 | COMP-24 IDP UAT token URI hardcoded as default in `application.yml` — only client-id and client-secret externalised | COMP-24 | Component file |
| RISK-076 | 🟡 | `spring-boot-devtools` shipping in production images — dev-time class-path scanning may execute in prod | COMP-23 (confirmed), COMP-24 (likely) | Component file |
| RISK-077 | 🟡 | COMP-43 / COMP-12 / COMP-17 / COMP-18 — outbox rows carry no Kafka-coordinate columns; incident correlation requires log-side join on `idempotencyId` | COMP-12, 17, 18, 41, 43 | Observability gap |
| RISK-078 | 🟡 | COMP-27 external VAP/LATAM auth fall-through — no entity-scope authorization service called | COMP-27 | OQ — architect intent |
| RISK-079 | 🟢 | COMP-17 `v-correlation-id` not propagated on IDP token call (only on case-search). Correlation chain breaks at IDP boundary | COMP-17 | Component file |
| RISK-080 | 🟢 | COMP-41 Spring Retry imports (`@Retryable`, `@Backoff`) are dead — never applied at runtime | COMP-41 | Dead code |
| RISK-081 | 🟢 | COMP-15 platform string uppercased on publish (`setPlatform(platform.toUpperCase())`) — protocol-contract drift if downstream relies on case sensitivity | COMP-15 | Contract-edge |
| RISK-082 | 🟢 | COMP-14 `auto.offset.reset = latest` — cold start with no committed offset SKIPS messages rather than replays | COMP-14 | DEC-005 sibling |

#### Reconciliation summary

- **Withdrawn (2026-04-23):** RISK-009 — underlying table absent from source.
- **Extended:** RISK-015 (extended by RISK-040 with 3 specific COMP-41 PUBLISHED-orphan paths).
- **Refined scope:** RISK-018 (COMP-19 / COMP-20 split-brain — see also RISK-028 NAP-specific MC CHI / AMEX / DISCOVER paths).
- **New ADR candidates:** RISK-028 (HIGH priority — formally accept or remediate AcceptService NAP split-brain).

---

### 6.2 Critical Risk Details (RISK-001 to RISK-004 — Phase 1)

---

**RISK-001: No circuit breakers on any external dependency**
**Severity:** 🔴 CRITICAL | **Reference:** DEC-014 VOID

Resilience4j is confirmed absent from all 40 WDP components. No circuit breaker, no rate limiter, and no bulkhead is configured on any outbound dependency anywhere in the platform. BusinessRulesProcessor (COMP-16) is the highest-risk instance: it calls seven downstream REST services with no timeout and no circuit breaker; consumer concurrency is 1 per replica, so a single hung thread stalls all `business-rules` message processing for that instance indefinitely with no automatic recovery. The same pattern repeats across all WDP components that make outbound REST calls. Reaffirmed 2026-04-25 — COMP-21 alone has 38 unprotected call sites across 12 target applications.

**NFR gap:** No timeout targets or acceptable hung-thread duration defined.

---

**RISK-002: No RestTemplate timeouts on any WDP component**
**Severity:** 🔴 CRITICAL | **Reference:** Confirmed across all component files

No WDP component configures explicit connection or read timeouts on `RestTemplate`. All outbound REST calls rely on OS-level TCP timeouts, which are effectively infinite under normal network conditions. A downstream service that accepts the TCP connection but never responds will hold the calling thread indefinitely.

⚠️ **Compounding finding (2026-04-18) — COMP-15 V3 PATCH:** The first V3 Update Action PATCH MUTATES the shared `RestTemplate` bean's request factory, applying 30s/30s timeouts to every subsequent REST call on the pod. Order-dependent global state.

**NFR gap:** No timeout threshold targets defined for any external dependency.

---

**RISK-003: At-most-once Kafka delivery — events lost on pod crash**
**Severity:** 🔴 CRITICAL | **Reference:** DEC-005

All WDP Kafka consumers commit the Kafka offset before processing begins. A pod crash after the commit and before processing completes permanently loses the event. There is no automatic redelivery, no Kafka DLQ, and no compensating mechanism.

⚠️ **Compounding finding (2026-04-18) — COMP-12 outbox→Kafka relay is the inverse pattern:** mark-and-send within `@Transactional` produces at-least-once with duplicate-possible window. Consumer-side `idempotency-key` dedup is the contracted mitigation.

⚠️ **Compounding finding (2026-04-25) — RISK-025:** The platform-wide empty `CommonErrorHandler{}` pattern is a *distinct* silent-loss class from RISK-003 — deserialisation failures and unhandled application exceptions are swallowed with no DLT, no log, no counter.

**NFR gap:** No acceptable event loss rate target defined.

---

**RISK-004: Clear PAN persisted on standard case creation (PostgreSQL)**
**Severity:** 🔴 CRITICAL | **Reference:** DEC-019

CaseManagementService (COMP-23) standard case creation writes the card number in clear text to `nap.case.I_ACCT_CDH` *(typo `I_ACCI_CDH` corrected 2026-04-23)* and `wdp.CASE.I_ACCT_CDH`. PAN encryption only occurs during the transaction enrichment flow, restricted to PIN and CORE platforms. Live PCI-DSS compliance gap.

**Sibling (2026-04-25) — see RISK-035:** COMP-43 writes clear PAN to `BC.TBC_DM_CASE.I_ACCT_CDH` on DB2 on CREATE+actionSeq=01 path.

**Accepted risk:** Formally recorded in DEC-019. Interim mitigation is database access controls. Remediation: move PAN encryption into the standard case creation transaction.

---

### 6.3 Critical Risk Details (RISK-024 onwards — Phase 2)

For brevity, Phase 2 critical risks not separately described below are documented in the summary table above with sufficient detail. The following are the highest-impact Phase 2 critical risks worth narrative emphasis.

---

**RISK-024: No K8s probes on multiple components — kubelet cannot evict hung pods**
**Severity:** 🔴 CRITICAL | **Reference:** Component files

Multiple long-running components have no liveness, readiness, or startup probes wired. The kubelet cannot detect a stalled JVM or hung listener thread. Pod readiness is governed only by `minReadySeconds: 30`. Combined with RISK-001 and RISK-002 (no timeouts, no circuit breakers), a stalled pod holds the consumer-group slot indefinitely without recovery.

Affected: COMP-07 VisaDisputeBatch, COMP-08 FirstChargebackBatch, COMP-09 CaseFillingBatch, COMP-11 FileProcessor (also no Actuator), COMP-12 InboundDisputeEventScheduler, COMP-14 CaseCreationConsumer (no liveness, no startup; readiness present), COMP-17 CaseExpiryUpdateConsumer, COMP-41 ThirdPartyNotificationConsumer (paths exposed at port 8082 but no probe block), COMP-43 CoreNotificationConsumer.

**NFR gap:** No platform-wide probe contract defined. ADR candidate.

---

**RISK-025: Empty `CommonErrorHandler{}` silent swallow — distinct silent-loss class**
**Severity:** 🔴 CRITICAL | **Reference:** Component files

Platform-wide pattern: empty anonymous `CommonErrorHandler{}` registered on consumer factories. Combined with `ErrorHandlingDeserializer`, deserialisation exceptions and unhandled application exceptions are silently dropped with no DLT, no log, no counter. This is a *distinct* silent-loss class from RISK-003 (pre-ACK offset window).

Affected: COMP-14, COMP-15, COMP-16, COMP-17, COMP-18, COMP-39, COMP-41, COMP-42, COMP-43.

**Variant — COMP-43 rebalance loop (RISK-057 🟡):** Bad payloads cause NPE at first event-field dereference; the outer try/catch swallows but no ACK fires. The message redelivers up to `max.poll.interval.ms` then the consumer is expelled from the group, triggering rebalance. Persistent bad payloads can produce a rebalance loop.

**NFR gap:** No platform-wide bad-payload contract defined. ADR candidate.

---

**RISK-028: COMP-19 NAP split-brain on MC CHI and AMEX/DISCOVER**
**Severity:** 🔴 CRITICAL | **Reference:** ADR pending

Two split-brain Kafka publish paths confirmed: (a) MC CHI on NAP — `MasterCardServiceImpl.accept` silently returns for CHI (treats as no-op); the Step 8 Kafka gate still fires if inbound actionCode is `FCHG/IPAB/IARB/IDCL`. (b) AMEX/DISCOVER on NAP — `cardNetwork` switch defaults to `log.warn`. Result on either path: NAPOutcomeProcessor delivers acceptance to NAP-DPS while the card network was never asked.

**Same severity class as DEC-019 / DEC-020.** Architect decision required: fail-close those branches before they reach the Kafka publish gate, OR formally accept the risk and document a manual recovery procedure for affected NAP cases.

---

**RISK-035: COMP-43 clear PAN persisted to BC.TBC_DM_CASE.I_ACCT_CDH (DB2)**
**Severity:** 🔴 CRITICAL | **Reference:** DEC-019 sibling (DB2)

CoreNotificationConsumer (COMP-43) on Step 7 CREATE + actionSeq=01 path decrypts HPAN via Encryption Service at the Step 6 PAN gate, then writes clear PAN to DB2 column `BC.TBC_DM_CASE.I_ACCT_CDH` *(corrected 2026-04-25 from `I_ACCT_CDR`)* and the last-4 to `I_ACCT_CDH_LST`.

**Architect decision required (OQ-COMP-43-6):** confirm whether this is intentional approved exception (document as DEC-019 extension) or remediate. Same severity class as RISK-004.

---

**RISK-038: COMP-12 replicas>1 unmitigated concurrency race**
**Severity:** 🔴 CRITICAL | **Reference:** DEC-023 extension

Source verification 2026-04-18 confirmed COMP-12 has **no `@SchedulerLock`, no advisory lock, no `SELECT FOR UPDATE`, no `synchronized` guard**. The XLD-templated replica count (`{{ replicas-wdp-chargeback-evidence-event-scheduler }}`) is not visible in source. Any production replica value > 1 produces guaranteed duplicate Kafka publishes — the five schedulers would each run their queries on every active replica.

**Action required:** Confirm production replica count via deployment infrastructure team. If > 1, immediate remediation (ShedLock or advisory lock) required.

---

### 6.4 High Risk Details (RISK-005 to RISK-012 — Phase 1)

[Phase 1 HIGH risks RISK-005 through RISK-012 detailed in v2.0 — content unchanged. See WDP-NFRS.md v2.0 narrative paragraphs.]

---

### 6.5 High Risk Details (RISK-044 onwards — Phase 2)

The Phase 2 HIGH risks are documented in the summary table above. The following warrant narrative emphasis.

---

**RISK-040: COMP-41 three distinct PUBLISHED-orphan paths**
**Severity:** 🔴 → noted under HIGH narrative for clarity; classification is CRITICAL | **Reference:** Extends RISK-015

Three distinct paths leave outbox rows at PUBLISHED (channel_type=GP_EVENTS) with no automatic recovery, **all invisible to COMP-12 Scheduler3 if Scheduler3 reads only FAILED/PENDING_DEFERRED**:

1. **Post-ACK crash before Signifyd response.** Offset committed to broker; pod dies; outbox row stays PUBLISHED forever.
2. **Signifyd empty body (`"NO_DATA_FROM_SIGNIFYD"`).** No status transition is performed — outbox row retains PUBLISHED status without any indication that processing did not complete.
3. **Final outbox UPDATE failure after ACK.** PostgreSQL unavailability or constraint failure leaves the row at PUBLISHED while the offset is committed.

OQ-COMP41-1: Confirm whether COMP-12 Scheduler3 ever reads PUBLISHED rows. If not, manual operator runbook is required.

---

**RISK-046: COMP-24 ActionEvent post-commit split-brain on EP 2/8/9**
**Severity:** 🟠 HIGH | **Reference:** DEC-001 PARTIAL

Source verification 2026-04-23 corrected the v1.0 claim that COMP-24 has full post-commit split-brain. The corrected scope:
- **BRE Kafka publish IS inside `@Transactional`** — Kafka failure rolls back the DB write. No split-brain on the BRE topic.
- **ActionEvent publish IS outside `@Transactional`** on EP 2 / 8 / 9 — domain commit succeeds; ActionEvent publish lost on broker failure when `napUpdateEvent=true`. **Genuine post-commit split-brain.**

DEC-001 deviation map updated to PARTIAL.

---

### 6.6 Medium and Low Risk Details (Phase 2)

For completeness, all Phase 2 MEDIUM and LOW risks are recorded with sufficient detail in the summary table at Section 6.1. Component-specific narrative is captured in the relevant component file (WDP-COMP-NN-*.md) and in WDP-HANDOVER.md "Component-Specific Source-Verified Findings".

---

### 6.7 Component-to-Risk Index (2026-04-25)

Navigation aid for finding all risks affecting a given component.

| Component | Phase 1 risks | Phase 2 risks |
|-----------|---------------|---------------|
| COMP-02 UAMS | RISK-010 | — |
| COMP-03 CHAS | RISK-012 | — |
| COMP-07 VisaDisputeBatch | RISK-013, 014 | RISK-024 |
| COMP-08 FirstChargebackBatch | RISK-013, 014 | RISK-024, 052, 061, 062 |
| COMP-09 CaseFillingBatch | RISK-013 | RISK-024, 053, 063 |
| COMP-11 FileProcessor | — | RISK-024, 054, 064, 065, 073, 074 |
| COMP-12 InboundDisputeEventScheduler | RISK-015 | RISK-024, 038, 077 |
| COMP-13 FileAcknowledgementProcessor | — | RISK-056 |
| COMP-14 CaseCreationConsumer | — | RISK-024, 025, 072, 082 |
| COMP-15 EvidenceConsumer | — | RISK-002 (compounding), 025, 029, 030, 081 |
| COMP-16 BusinessRulesProcessor | RISK-007, 008, 011, 020, 022 | RISK-025 |
| COMP-17 CaseExpiryUpdateConsumer | — | RISK-024, 025, 031, 045, 077, 079 |
| COMP-18 NotificationOrchestrator | RISK-015 | RISK-025, 077 |
| COMP-19 AcceptService | RISK-018 | RISK-028, 044 |
| COMP-20 ContestService | RISK-018 | RISK-044 (likely shared) |
| COMP-21 ChargebackService | — | RISK-026, 027, 049, 050, 051, 070, 071 |
| COMP-22 DisputeService | RISK-021 | — |
| COMP-23 CaseManagementService | RISK-004, 006 (RISK-009 WITHDRAWN) | RISK-032, 033, 067, 068, 069, 076 |
| COMP-24 CaseActionService | RISK-005 | RISK-039, 046, 047, 048, 075, 076 |
| COMP-25 NotesService | — | — |
| COMP-26 QuestionnaireService | — | — (existing OQ on POST idempotency gap) |
| COMP-27 CaseSearchService | — | RISK-041, 042, 043, 078 |
| COMP-28 DisplayCodeService | RISK-023 | — |
| COMP-34 MerchantTransactionService | RISK-019 | — |
| COMP-37 DocumentManagementService | — | RISK-034, 066 |
| COMP-39 NAPOutcomeProcessor | RISK-016 | RISK-025 |
| COMP-40 VisaResponseQuestionnaire | RISK-017 | — |
| COMP-41 ThirdPartyNotificationConsumer | — | RISK-024, 025, 040, 055, 080 |
| COMP-42 BENConsumer | — | RISK-025 |
| COMP-43 CoreNotificationConsumer | — | RISK-024, 025, 035, 036, 037, 057, 058, 059, 060, 077 |

---

*This document covers non-functional requirements and confirmed platform risks.*
*v2.1 reconciled 2026-04-25 — 60 new RISK rows added, 1 withdrawn (RISK-009), 2 extended (RISK-015, RISK-018).*
*NFR targets for unaddressed risks to be set by product team.*
*Implementation details, database schemas, and deployment specifications are maintained in individual component files (WDP-COMP-[NN]-*.md).*
