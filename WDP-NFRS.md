# WDP-NFRS.md
**Worldpay Dispute Platform — Non-Functional Requirements**
*Version: 2.2*

---

## How to Read This Document

NFRs are organised into five confirmed sections (Performance, Availability and Resilience, Security and Compliance, Scalability, Operational Constraints) followed by Section 6: the Platform Risk Register.

**Sections 1–5** carry forward NFR targets from v1.0/v2.0/v2.1, with three categories of correction applied:
- Removed entries that referenced Resilience4j circuit breakers (DEC-014 VOID) or BRE step checkpointing (DEC-011 VOID)
- Corrected the Kafka delivery guarantee claim (at-most-once, not at-least-once)
- Added exception notes where confirmed component behaviour conflicts with a stated NFR

**Section 6 — Platform Risk Register.** The v2.0 register documented 23 confirmed risks (RISK-001 through RISK-023). The v2.1 reconciliation appended RISK-024 through RISK-082 from the 20-entry source-verification audit and withdrew RISK-009. **The v2.2 reconciliation appends RISK-083 onwards from the 18-entry source-verification audit**, extends the platform-wide pattern rows (RISK-001, 010, 012, 013, 024, 025, 040, 050, 076, 077) with newly-confirmed components, and promotes RISK-010 from 🟠 to 🔴 to reflect the DEC-021 scope expansion.

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
| File upload → outbox insert | < 30 seconds | Target | Per file | |
| File upload → outbox insert | < 45 seconds | P95 | Per file | |
| File upload → outbox insert | < 60 seconds | P99 | Per file | |
| Outbox insert → Kafka publish | < 2 minutes | Target | Per batch | |
| Outbox insert → Kafka publish | < 3 minutes | P95 | Per batch | |
| Outbox insert → Kafka publish | < 5 minutes | P99 | Per batch | |
| Kafka publish → evidence attached | < 30 seconds | Target | Per row | |
| Kafka publish → evidence attached | < 60 seconds | P95 | Per row | |
| Kafka publish → evidence attached | < 120 seconds | P99 | Per row | |
| Full evidence flow end-to-end | < 3 minutes | Target | | |
| Full evidence flow end-to-end | < 5 minutes | P95 | | |
| Full evidence flow end-to-end | < 10 minutes | P99 | | |

⚠️ OUTDATED — An earlier document states ACK generation target of < 60 seconds at P99. The production SLO document does not repeat this figure and instead defines ACK generation success rate (> 95%) as the primary SLO. Verify which metric governs ACK SLA in production.

⚠️ NEW — File job COMPLETED transition ownership undocumented (RISK-088). COMP-11 never writes COMPLETED; COMP-13 polls for `status IN (COMPLETED, ERROR)`; COMP-12 is the candidate writer per WDP-DB.md but the contract is undocumented in all three repos. ACK-generation SLO measurement is uncertain pending contract confirmation.

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

⚠️ EXCEPTION — COMP-21 ChargebackService throughput ceiling is far below replica-count expectations. The `asyncExecutor` is core=1, max=1, queue=5; the parallel ACL+lookup pattern provides no real parallelism. Seventh concurrent action request hits `RejectedExecutionException`. See RISK-026.

⚠️ EXCEPTION — **COMP-26 QuestionnaireService** uses `DriverManagerDataSource` (no HikariCP, no application-tier connection pool). Every JPA call opens a new JDBC connection — throughput is bounded by Postgres / PgBouncer accept rate. See RISK-147.

⚠️ EXCEPTION — **COMP-32 RulesService** uses `DataSourceBuilder.create.build` with no explicit pool tuning on either of two PostgreSQL datasources. Pool sizing is whatever Spring Boot defaults to. See RISK-163 (DB outage takes down all 14 endpoints simultaneously due to absent timeouts/retry/fallback).

---

### 1.3 API Response Latency (Stage 1 and Stage 2)

| Requirement | Target | Percentile | Source |
|---|---|---|---|
| Internal API response time | < 200 ms | P95 | Cross-cutting target |
| Notification delivery from case state change | < 30 seconds | | Stage 1 design target |
| Encryption API — encrypt | < 50 ms | P95 | Excl. rare DEK refresh |
| Encryption API — decrypt | < 75 ms | P95 | Incl. KMS call when DEK cached |
| WDP Portal — Merchant mode — queue list load | < 1 second | | Stage 2 UI target |
| WDP Portal — Merchant mode — queue case list load | < 2 seconds | | Stage 2 UI target |
| WDP Portal — Merchant mode — dispute detail load | < 2 seconds | | Stage 2 UI target |
| WDP Portal — Merchant mode — queue refresh | < 500 ms | | Stage 2 UI target |
| Dispute list — page load | < 5 seconds | P95 | Stage 2 UI alert threshold |
| Dispute list — search API | < 10 seconds | P95 | Stage 2 UI alert threshold |

Note: v1.0 contained a row "BRE validation per step — < step-specific timeout." This referenced BRE step checkpointing (DEC-011), which is void — the named steps do not exist in the BusinessRulesProcessor codebase. This row has been removed.

⚠️ EXCEPTION — COMP-21 ChargebackService is the externally-visible WDP entry point for merchants. Its `contest` WDP-path performs ~10 sequential round-trips (5 business calls each preceded by a fresh IDP token GET, no token cache, no connection pool) on a bare `RestTemplate`. This sets a hard latency floor on the platform's externally-visible API. See RISK-025 (legacy numbering; v2.1 doc) — *cross-reference confirmed correct*.

⚠️ EXCEPTION — **COMP-35 EncryptionService decrypt latency target (< 75 ms P95) is at risk** because `@Transactional` brackets the KMS network call: a Hikari connection is held during the KMS round-trip on every decrypt request. Sustained decrypt load combined with KMS slowdown can exhaust the pool. See RISK-172.

⚠️ EXCEPTION — **COMP-01 API Gateway latency target (P95 < 200 ms internal) is at risk** because the gateway uses a blocking `RestTemplate` on the Netty event-loop for UAMS / CHAS calls, with no timeout on either dependency. A single degraded auth service can exhaust all gateway threads. See RISK-093.

---

### 1.4 Stage 3 Analytics Performance

All Stage 3 targets are from planning documents only. None have been measured.

| Requirement | Target | Percentile |
|---|---|---|
| ETL job completion (daily load) | < 2 hours | For 1M cases/day |
| ETL data freshness | < 24 hours | P95 |
| ETL failure rate | < 1% | |
| Simple query (pre-built reports) | < 2 seconds | P95 |
| Complex query (custom reports) | < 10 seconds | P95 |
| Dashboard load time | < 3 seconds | P95 |
| ML prediction latency | < 100 ms | P95 |
| Report API response | < 500 ms | P95 |
| Data export generation (10k rows) | < 30 seconds | |
| Concurrent analytics users supported | 1,000+ | |

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

⚠️ EXCEPTION — Pod availability for several long-running components is not measurable in the standard sense because they have no Kubernetes liveness, readiness, or startup probes. Hung pods are not evicted by kubelet. Affected (extended): COMP-07, COMP-08, COMP-09, COMP-11, COMP-12, COMP-14, COMP-17, COMP-39 (no startup), COMP-41, COMP-42 (no probes at all), COMP-43, COMP-51 (no probes at all). See RISK-024.

⚠️ EXCEPTION — **`minReadySeconds` platform-wide misplacement.** Eight components confirmed to place `minReadySeconds: 30` under `spec.template.spec` instead of `spec` — silently ignored at runtime. Affected: COMP-03, COMP-05, COMP-08, COMP-09, COMP-12, COMP-25, COMP-26, COMP-28, COMP-34, COMP-40. The intended rolling-update stability gate is not actually applied. See RISK-083.

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

### 2.3 Resilience Behaviour

The v1.0 document described circuit breaker thresholds for Document Management, card network integrations, and the Encryption API. **All circuit breaker entries have been removed.** DEC-014 is void — Resilience4j confirmed absent from all 38 source-verified WDP components.

The v1.0 delivery guarantee claim "At-least-once; offset committed only after full processing" has been corrected. The platform uses at-most-once delivery — see DEC-005.

| Requirement | Confirmed Specification |
|---|---|
| Kafka consumer delivery guarantee | **At-most-once** on Kafka consumers (COMP-14/15/16/17/18/39/40/41/42/43). **At-least-once with duplicate-possible window** on COMP-12 outbox→Kafka relays (mark-and-send within `@Transactional`; broker ACK precedes TX commit). Consumer-side `idempotency-key` dedup is the contracted mitigation. ⚠️ COMP-05 added to the at-most-once list explicitly — pre-ACK confirmed in source. |
| Retry mechanism for external calls | Spring Retry (`@Retryable`) — present in a subset of components only. Typically 3 attempts with fixed delay. COMP-34 has no retry. ⚠️ COMP-41 imports `@Retryable`/`@Backoff` but never applies them — class names containing "Retry" describe custom try/catch, not framework. ⚠️ COMP-42 spring-retry is transitive via spring-kafka — not declared in pom.xml; same risk class as COMP-41. ⚠️ COMP-51 hand-rolls retry via `i_retry_count` increment on `wdp.case_expiry`; spring-retry is on classpath but unused. |
| Outbox retry — transient failures | At least 2 retry attempts tracked via `wdp.outgoing_event_outbox` status progression before terminal ERROR (COMP-43 pattern). Not universally implemented. ⚠️ Scheduler3 reads only FAILED and PENDING_DEFERRED rows — PUBLISHED orphans have no auto-redrive. See RISK-015 / RISK-040. ⚠️ COMP-51 EXPIRY_BATCH rows are written at `status=ERROR` direct (no PUBLISHED→FAILED transition); **no platform component is currently confirmed to read them**. See RISK-090. |
| Notification channel isolation | Per-channel outbox table; failure in one channel (e.g. CORE_EVENTS) does not affect other channels (e.g. BEN, Signifyd). ⚠️ Now five channels confirmed: EXPIRY_EVENTS, GP_EVENTS, BEN_EVENTS, CORE_EVENTS, EXPIRY_BATCH. |
| ACK generation success rate SLO | > 95% rolling 1 hour; alert at < 90% (CRITICAL) — ⚠️ measurement reliability depends on COMPLETED-transition contract (see Section 1.1) |
| Kafka broker durability | 3 brokers, 3 AZs; acks=all; min in-sync replicas = 2; idempotent producer enabled |
| Bad-payload handling | ⚠️ Empty `CommonErrorHandler{}` registered platform-wide on multiple consumers — silent drop on deserialisation failure with no DLT, no log, no counter. Distinct silent-loss class. See RISK-025. ⚠️ **Affected list extended** to include COMP-05, COMP-40, COMP-42 (now 10 components confirmed). |

**What no longer applies (removed from v1.0):**
- Document Management API circuit breaker thresholds
- Network integration circuit breaker thresholds
- Encryption API circuit breaker scope
- DEK cache window on KMS outage (6-hour figure — **now confirmed incorrect** per COMP-35 audit; actual rotation interval is **days** via `${dek_rotation_interval_days}`)
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

⚠️ EXCEPTION — Producer idempotence guarantees only-once delivery within a single producer session for retries on the same `send` call. **It does not span separate inbound HTTP requests.** COMP-04 has no application-level inbound idempotency — duplicate POSTs from clients produce duplicate Kafka events regardless of broker idempotence. See RISK-118.

---

## 3. Security and Compliance

### 3.1 Compliance Frameworks

| Framework | Scope | Status |
|---|---|---|
| PCI-DSS 3.2.1 | Full platform — all cardholder data flows | ✅ Active — ⚠️ **NEW MATERIAL DEFICIENCY:** see RISK-084 (CVV at rest in COMP-05 + CVV in logs in COMP-04) |
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
| PAN in logs | Must never appear in application logs, metrics, or traces ⚠️ **EXCEPTION** — see RISK-114, RISK-116 (COMP-04 base64 file content + CVV via Lombok toString) |
| PAN in ACK files | ACK files must use merchant-provided chargeback identifier, not PAN |
| Encryption algorithm | AES-256-GCM |
| Tokenisation algorithm | HMAC-SHA256 (for HPAN — deterministic, non-reversible) |
| Key storage | AWS KMS with FIPS 140-2 Level 3 validated HSMs; keys never leave HSM |

⚠️ EXCEPTION — DEC-019 (PostgreSQL): CaseManagementService (COMP-23) standard case creation (`POST /{platform}/case`) writes clear PAN to `nap.case.I_ACCT_CDH` and `wdp.CASE.I_ACCT_CDH` before encryption occurs. PAN encryption only takes place during the transaction enrichment flow, which is a secondary path restricted to PIN and CORE platforms. Database access controls are the interim mitigation. See RISK-004.

⚠️ EXCEPTION — DEC-019 (DB2): **** CoreNotificationConsumer (COMP-43) writes clear PAN to `BC.TBC_DM_CASE.I_ACCT_CDH` on Step 7 CREATE + actionSeq=01 path after Encryption Service decrypt at the Step 6 PAN gate. Whether this is intentional and approved is owed by the CORE platform team. See RISK-035.

⚠️ EXCEPTION — In-flight PAN surface: **** COMP-21 ChargebackService surfaces full `cardNumber` in two response model classes (`SearchCaseList`, `Transaction`) and `cardNumberLast4` in eight others. See RISK-051.

⚠️ EXCEPTION — DEC-004 edge case: **** COMP-11 `NetworkFileSupport` encrypts only when `acctNum` matches `\d+`. The non-numeric branch passes the raw value into `chbk_outbox_row.payload.account_number`. See RISK-073.

⚠️ EXCEPTION — 🔴 **CVV at rest (NEW PCI-DSS):** **COMP-05** persists CVV in two places per error row in `NAP.DISPUTE_EVENT_CONSUMER_ERROR` — `C_CVV` column directly mapped on the entity, AND inside `C_KAFKA_EVENT` raw JSON which retains the inbound `cvv` field. Cross-link: **COMP-04** logs the CVV via Lombok-generated `NapEvent.toString` at INFO level immediately before Kafka publish. Together these constitute a CVV-on-disk path. **PCI-DSS 3.2.1 Requirement 3.2 prohibits storing CVV after authorisation under any circumstance.** Architect decision required: remediate (clear `C_CVV` column on insert; redact `C_KAFKA_EVENT` JSON; remove `toString` from logging surface) or document approved exception. See RISK-084.

---

### 3.3 Key Management Constraints

| Requirement | Specification |
|---|---|
| Master key (KEK) location | AWS KMS HSM, FIPS 140-2 Level 3 validated |
| KEK rotation | Annual via AWS KMS automatic rotation |
| Data encryption keys (DEK) rotation | **Days, configured via `${dek_rotation_interval_days}`** ** |
| DEK derivation | KMS-protected; encrypted DEKs stored in `wdp.data_enc_key`; KMS round-trip required to decrypt DEK before each encryption operation in COMP-35 |
| Key access | KMS IAM policy allows EncryptionService service account only |
| Crypto audit | All encrypt and decrypt operations logged in `crypto_audit` table with timestamp and component name |
| HPAN deterministic hash | HMAC-SHA256, salted with rotating HMAC key from `wdp.hash_key_store` |

⚠️ EXCEPTION — **COMP-35 has no method-level authorization on the decrypt endpoint.** Decrypt-caller policy (COMP-14 only) is convention, not technical control. Any service that can reach the decrypt endpoint and obtain a valid JWT can decrypt. See RISK-176-class operational risk.

⚠️ EXCEPTION — **COMP-35 `rotateDEK` reuse-existing-DEK branch carries a Base64 bug.** `dek_enc` is passed to KMS as raw UTF-8 bytes of the Base64 string instead of decoded ciphertext. Symptom only manifests on pod restart that finds a recent-enough DEK row to reuse. Failure is silently swallowed; pod starts with `initialized=false`. See RISK-171.

⚠️ EXCEPTION — **COMP-35 multi-pod DEK rotation race.** No distributed lock; concurrent pods may attempt rotation simultaneously. See RISK-173.

---

### 3.4 Authentication and Authorization

| Requirement | Specification |
|---|---|
| External merchant auth | OAuth 2.0 — JWT issued by IDP service (SunGard for NAP) |
| Internal user auth | OAuth 2.0 — same IDP service |
| Token validation | JWT signature, issuer, expiry, scopes — all validated at API Gateway |
| Token expiry | 1 hour for access tokens; 24 hours for refresh tokens |
| Merchant data isolation | Row-level isolation by merchant_id enforced at API level |
| Merchant API — additional auth layer | API Key required in addition to OAuth JWT for external merchants |
| Merchant API — rate limit | 1,000 requests per hour per merchant |
| Operations user roles | Tier 1 Agent, Tier 2 Agent, Team Lead, Manager, Admin — with distinct action permissions |
| Queue access | Users may only access disputes in queues assigned to their role |
| Operation-level RBAC | ⚠️ EXCEPTION — DEC-018: RBAC enforcement not active in CaseActionService (COMP-24). See RISK-005. |
| Internal vs external authorization | ⚠️ EXCEPTION — COMP-27 CaseSearchService `POST /lft` derives internal-vs-external authorization from a request-body field (`LftSearchParams.isInternal`), NOT from the JWT. See RISK-042. |
| Org-scope authorization on CHAS | ⚠️ EXCEPTION — `validateOrgId` commented out in COMP-03 on `GET /orgentity`. The method body is itself absent from `RequestValidator` — remediation requires reimplementation, not uncomment. See RISK-012. |
| External entity-scope authorization | ⚠️ EXCEPTION — External VAP and LATAM callers in COMP-27 are not routed through any entity-scope authorization service. Only NAP (→ UAMS) and PIN/CORE (→ CHAS) are. Intent unconfirmed. |
| CORE/VAP/LATAM gateway-level authorization | ⚠️ EXCEPTION — **No CORE platform value exists in COMP-01 source code.** CORE/VAP/LATAM requests receive no role-level or case-level authorization at the gateway layer. Architect decision required. |
| Internal PIN regular role | **** `WDP_PIN_REGULAR` role exists in `AuthorizationList` alongside `WDP_NAP_REGULAR`. |
| Partner identity routing | **** COMP-21 ChargebackService identifies partners via `entitlement_params` consumer name: `SIGNIFYD` and `JUSTTAI` (double-T). ACL chain-ID validation is skipped for partners. |
| COMP-04 inbound auth | ⚠️ EXCEPTION — All endpoints unauthenticated at app level; SecurityConfig whitelist `/**`. Auth relies entirely on Ingress / network controls. See RISK-115. |
| COMP-22 internal-firm enforcement | ⚠️ EXCEPTION — application-layer (not Spring Security), `contains` check on JWT `iss` claim against literal `us_worldpay_fis_int`. `ForbiddenException` constructor uses `HttpStatus.UNAUTHORIZED` but global handler returns 403 — constructor's status code is dead. See RISK-138. |
| CHAS internal firm bypass scope | **** Internal firm bypass (`iss` contains `us_worldpay_fis_int`) applies to BOTH `/authorize` AND `/entity-authorize` — both share `AuthorizationServiceImpl.authorizeEntity`. Partial bypass on `POST /{platform}/merchant/entitytype` (V1 and V2): internal callers skip JWT `iqentities` extraction and use `orgId` from request body directly. |
| `/actuator/prometheus` JWT-protection | ⚠️ EXCEPTION — Confirmed widespread. COMP-21, COMP-25, COMP-29, COMP-35 all require JWT on Prometheus scrape — scrape side configuration must carry JWT or platform-wide whitelist remediation needed. See RISK-140 (COMP-25 specific) and observability gap class. |

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

⚠️ EXCEPTION — Logstash audit-pipeline reliability: COMP-21 production secret `logstash_server_host_port` is empty. Only stdout logs reach aggregation today. See RISK-050.

⚠️ EXCEPTION — **COMP-30 `logback-spring.xml` absent from repo.** Logstash integration is non-functional unless mounted via ConfigMap/sidecar at runtime. Same pattern class as RISK-050. See RISK-161.

---

### 3.6 Stage 3 Security Constraints

(Unchanged from v2.1.)

---

## 4. Scalability

### 4.1 Horizontal Scaling Boundaries

| Component | Min Instances | Max Instances | Scale Trigger |
|---|---|---|---|
| File Processor (COMP-11) | 3 | 10 | CPU > 70% or memory > 80% |
| Publisher Scheduler (COMP-12) | 2 | 2 (fixed) | Leader election; no further horizontal scaling |
| Chargeback Worker ⚠️ VERIFY | | | Kafka consumer lag > 1,000 per pod |
| Evidence Worker (COMP-15 EvidenceConsumer) | 5 | 20 | Kafka consumer lag > 1,000; max capped by Kafka partition count |
| EKS cluster nodes | 10 | 50 | CPU/memory utilisation; auto-scaling |

**Publisher Scheduler (COMP-12) constraint:** ⚠️ EXCEPTION — Source verification reveals **no `@SchedulerLock`, no advisory lock, no `SELECT FOR UPDATE`, no `synchronized` guard**. Replicas > 1 is a 🔴 unmitigated concurrency race. See RISK-038.

**Additional confirmed scaling constraints (extended for v2.2):**

| Component | Constraint | Reason |
|---|---|---|
| VisaDisputeBatch (COMP-07) | Replica must equal exactly 1 | Parallel replicas poll the same external queue — see DEC-023 |
| FirstChargebackBatch (COMP-08) | Replica must equal exactly 1 | Same reason — DEC-023 |
| CaseFillingBatch (COMP-09) | Replica must equal exactly 1 | Same pattern as COMP-07/08 — DEC-023 |
| **NAPDisputeDeclineBatch (COMP-06)** | **Replica must equal exactly 1** | **** Same scheduler-replica pattern as COMP-07/08; no `@SchedulerLock`, no advisory lock, no Redis lock. See RISK-129-class extension to RISK-013. |
| FileProcessor (COMP-11) | Per-pod `maxConcurrentMessages=1` | SQS-listener serialisation; cross-pod race on `(file_name, s3_key)` mitigated only by SQS message visibility |
| BusinessRulesProcessor (COMP-16) | Kafka consumer concurrency = 1 per replica | Single-threaded consumer; hung thread stalls all processing for that instance |
| CaseExpiryUpdateConsumer (COMP-17) | Concurrency = 1 per replica | No singleton guard; replicas > 1 rely entirely on Kafka consumer-group rebalance |
| ThirdPartyNotificationConsumer (COMP-41) | Concurrency = 1, no `setConcurrency` | Same pattern; auto.offset.reset=latest skips backlog on cold start |
| BENConsumer (COMP-42) | Concurrency = 1 (default) | Same pattern |
| CoreNotificationConsumer (COMP-43) | Concurrency = 1 (default), `max.poll.records=500`, `max.poll.interval.ms=600000` | Bad-payload rebalance loop possible — see RISK-068 |
| ChargebackService (COMP-21) | Per-pod `asyncExecutor` core=1, max=1, queue=5 | 7th concurrent action request hits `RejectedExecutionException`. See RISK-026. |
| **CaseExpiryProcessor (COMP-51)** | **Replica must equal exactly 1** | **** No `@SchedulerLock`, no ShedLock, no advisory lock. DEC-023 operational-only. Reader half of Case Expiry Subsystem. |
| **NAPOutcomeProcessor (COMP-39)** | **Concurrency = 1** | **** Intentional or default-by-omission unconfirmed — open question. |

---

### 4.2 Kafka Partition Constraints

(Unchanged content from v2.1.)

| Constraint | Value | Implication |
|---|---|---|
| wdp.file.outbox.events — partition count | 6 | Maximum effective parallel consumer instances = 6 without repartitioning |
| Partition key — stated standard | merchantId | All events for a merchant go to the same partition |
| Partition key — confirmed deviation | `caseNumber` used by all six business-rules publishers (COMP-12, 15, 23, 24, 25, 37); pass-through `KafkaHeaders.RECEIVED_KEY` on COMP-18 outbound topics; compound `networkCaseId+cardNetwork+platform` on COMP-12 → COMP-14 path; **COMP-04 partition key is `merchantId` for case-update and outcome paths but `cardAcceptorCodeId` on POST /event new-dispute path** | See DEC-003 deviation map in WDP-DECISIONS.md |
| Top-5 merchant volume share | ~40% of total | Hot partitions; monitor lag |
| Partition count change | Requires topic recreation or manual rebalancing | Treat partition count as semi-permanent |

---

### 4.3 Database Scalability

| Requirement | Specification |
|---|---|
| Aurora read replicas — minimum | 2 (production) |
| Aurora read replicas — auto-scale maximum | 5 |
| Auto-scale trigger | CPU > 70% or connection count > 700 |
| Scale-in cooldown | 5 minutes |
| Scale-out cooldown | 60 seconds |
| Connection pool maximum per service | 20 connections (HikariCP) ⚠️ EXCEPTION — COMP-22 HikariCP pools are unconfigured on both datasources; Spring Boot defaults apply (`maximumPoolSize=10`). COMP-26 uses `DriverManagerDataSource` (no pool at all). COMP-32 uses `DataSourceBuilder.create.build` with no explicit pool tuning. See RISK-132 / RISK-147 / RISK-163. |

---

### 4.4 MSK Storage Constraint — Permanent Floor

(Unchanged from v2.1.)

---

### 4.5 Stage 3 Scalability

(Unchanged from v2.1.)

---

## 5. Operational Constraints

### 5.1 Deployment Windows

(Unchanged from v2.1.)

⚠️ EXCEPTION — Several production component images ship `spring-boot-devtools` (COMP-23 confirmed; COMP-24 likely; **COMP-30 confirmed**). Dev-time class-path scanning may execute in production. See RISK-076.

---

### 5.2 Incident Response Constraints

(Unchanged content table from v2.1.)

⚠️ EXCEPTION — Incident correlation gaps (extended):
- COMP-12 Scheduler3/Scheduler4 paths do not persist Kafka metadata (`kafka_offset`, `kafka_partition`, `kafka_topic`) to outbox rows.
- COMP-43 outbox row entity carries no Kafka coordinate columns.
- COMP-17 `v-correlation-id` is not propagated on the IDP token call (only on case-search call). See RISK-079.
- COMP-43 has no MDC enrichment, no custom Micrometer metrics.
- **** **COMP-51** generates a fresh random `v-correlation-id` UUID per outbound REST call — same anti-pattern as COMP-17. End-to-end audit trail across the 4–5 calls per record cannot be reconstructed from headers alone. See RISK-089.
- **** **COMP-22** does not propagate `v-correlation-id` on any outbound REST call. The interceptor places it in MDC for local logs only. See RISK-136.
- **** **COMP-01** `v-correlation-id` always null on UAMS/CHAS calls — `RequestCorrelation.setId` never called; `ThreadLocal` incompatible with reactive WebFlux threading model. See RISK-094.

---

### 5.3 Data Retention and Archival

(Unchanged from v2.1 plus the COMP-08 SKIPPED-marker note.)

---

### 5.4 Monitoring Coverage Requirements

(Unchanged content table from v2.1.)

⚠️ EXCEPTION — Several components have no `management:` block in YAML profiles, exposing Spring Boot Actuator defaults only. Affected (extended): COMP-43, COMP-17, COMP-18 (default exposure only), **COMP-40** (Prometheus tag `${app.name}` referenced but never defined — resolves to empty string).

⚠️ EXCEPTION — **COMP-39 probe path mismatch:** `resources.yaml` probes hit `/merchant/gcp/update-consumer/nap/livez` but actuator `additional-path: server:/livez` exposes the endpoint at server root. Probe resolution requires runtime observation. See RISK-181.

---

### 5.5 Backup and Verification

(Unchanged from v2.1.)

---

### 5.6 Infrastructure Region Requirements

(Unchanged from v2.1.)

⚠️ NOTE — COMP-11 `S3ClientConfiguration` hardcodes AWS region `us-east-2` (not `us-east-1`). Reconcile against infrastructure documentation.

---

## 6. Platform Risk Register

This section documents confirmed gaps and risks. RISK-001 through RISK-023 were identified during the April 2026 component survey. RISK-024 through RISK-082 were added during the source-verification reconciliation pass. **RISK-083 through RISK-194 were added during the source-verification reconciliation pass against 16 additional components.** **RISK-195 through RISK-200 were added in the 2026-05-06 post-baseline reconciliation against COMP-49 WDP Portal.**

NFR targets for addressing these risks have not been set. Product team to define.

**Severity key:**
- 🔴 CRITICAL — data loss, compliance failure, or silent split-brain possible
- 🟠 HIGH — significant operational risk; partial data integrity or security gap
- 🟡 MEDIUM — operational friction or incomplete feature with known workaround
- 🟢 LOW — informational; minor inconsistency or dead code with no current impact

---

### 6.1 Risk Summary Table

#### Phase 1 — April 2026 component survey (RISK-001 to RISK-023)

(Table unchanged from v2.1 — see WDP-NFRS.md for the full Phase 1 listing. Reproduced abridged for reference; RISK-009 remains WITHDRAWN.)

| Risk ID | Severity | Risk | Affected | Reference |
|---|---|---|---|---|
| RISK-001 | 🔴 | No circuit breakers on any external dependency | All 38+ source-verified components | DEC-014 VOID |
| RISK-002 | 🔴 | No RestTemplate timeouts | All components with REST | Component files |
| RISK-003 | 🔴 | At-most-once Kafka delivery | All Kafka consumers | DEC-005 |
| RISK-004 | 🔴 | Clear PAN persisted on standard case creation (PostgreSQL) | COMP-23 | DEC-019 |
| RISK-005 | 🟠 | RBAC not enforced in CaseActionService | COMP-24 | DEC-018 |
| RISK-006 | 🟠 | No idempotency on case creation | COMP-23 | DEC-020 |
| RISK-007 | 🟠 | BRE inconsistent failure handling | COMP-16 | Component file |
| RISK-008 | 🟠 | BRE error visibility via SNOTE only | COMP-16 | Component file |
| RISK-009 | ⚠️ WITHDRAWN | | | |
| **RISK-010** | **🔴 (PROMOTED)** | **DEC-021 wrong-TM scope expansion — UAMS now confirmed 7 methods across 4 NAP tables (was 1 method); COMP-30 added as second offender** | **COMP-02, COMP-30** | **DEC-021** |
| RISK-011 | 🟠 | BRE split-brain on Kafka publish failure | COMP-16 | DEC-001 deviation |
| RISK-012 | 🟠 | validateOrgId commented out — **** method body absent; remediation cannot be done by uncomment alone | COMP-03 | DEC-018 (related) |
| RISK-013 | 🟡 | Polling batch replica constraint has no automated enforcement — extended to include **COMP-06** | COMP-06, 07, 08, 09, 51 | DEC-023 |
| RISK-014 | 🟡 | removeItemFromQueueDisabled has no automated state check | COMP-07, COMP-08 | DEC-022 |
| RISK-015 | 🟡 | bre_orchestration_outbox PUBLISHED orphan rows | COMP-12, COMP-18 | Component file |
| RISK-016 | 🟡 | NAPOutcomeProcessor notesLookup commented out | COMP-39 | Component file |
| RISK-017 | 🟡 | VisaResponseQuestionnaire additionalImagesList silently discarded | COMP-40 | Component file |
| RISK-018 | 🟡 | AcceptService/ContestService — no rollback on Kafka publish failure | COMP-19, COMP-20 | DEC-001 deviation |
| RISK-019 | 🟡 | MerchantTransactionService — no retry on any of 10 external dependencies | COMP-34 | Component file |
| RISK-020 | 🟢 | LATAM platform silently dropped in BusinessRulesProcessor | COMP-16 | Component file |
| RISK-021 | 🟢 | DisputeService Kafka producer wired but commented out | COMP-22 | Component file |
| RISK-022 | 🟢 | BRE source field routing not implemented | COMP-16 | Component file |
| RISK-023 | 🟢 | DisplayCodeService does not determine TIER1 eligibility | COMP-28 | Component file |

#### Phase 2 — source-verification reconciliation (RISK-024 to RISK-082)

(Table unchanged from v2.1 — RISK-024 through RISK-082 reproduced unmodified. See v2.1 for the full listing.)

#### Phase 3 — source-verification reconciliation (RISK-083 onwards)

##### 6.1.A Cross-Component / Platform-Wide additions

| Risk ID | Severity | Risk | Affected Components | ADR / Reference |
|---|---|---|---|---|
| RISK-083 | 🟠 | `minReadySeconds: 30` misplaced under `spec.template.spec` instead of `spec` — silently ignored by Kubernetes; intended rolling-update stability gate not actually applied | COMP-03, 05, 08, 09, 12, 25, 26, 28, 34, 40 | Pattern audit — DevOps remediation pass candidate |
| RISK-084 | 🔴 | **CVV at rest (PCI-DSS 3.2.1 prohibition)** — COMP-05 persists CVV in `NAP.DISPUTE_EVENT_CONSUMER_ERROR.C_CVV` AND in `C_KAFKA_EVENT` raw JSON; COMP-04 logs CVV via Lombok-generated `NapEvent.toString` at INFO before Kafka publish | COMP-04, COMP-05 | ADR pending — DEC-019-class |
| RISK-085 | 🔴 | **Cross-component shared-error-table consumption hazard on `NAP.DISPUTE_EVENT_CONSUMER_ERROR`** — both COMP-05 and COMP-39 prior-error scans query without `C_EVENT_TYPE` filter; rows written by either consumer (plus COMP-23, COMP-24) may be picked up and reprocessed through the wrong outbound pipeline | COMP-05, COMP-39 | ADR pending — uniform decision required |
| RISK-088 | 🟡 | File job COMPLETED transition ownership undocumented — COMP-11 never writes COMPLETED; COMP-13 polls for `status IN (COMPLETED, ERROR)`; COMP-12 is the candidate writer per WDP-DB.md but the contract is undocumented in all three repos *(forward-referenced from Section 1.1; formally numbered here)* | COMP-11, COMP-12, COMP-13 | Component contract gap |
| RISK-089 | 🟠 | **`v-correlation-id` regenerated-per-call anti-pattern** — COMP-17 and COMP-51 generate a fresh random UUID per outbound REST call rather than propagating the ingress correlation ID; end-to-end audit trail cannot be reconstructed from headers. COMP-22 fails to propagate v-correlation-id on any outbound REST call; COMP-01 always sets it null. | COMP-01, COMP-17, COMP-22, COMP-51 | Platform-wide remediation candidate |
| RISK-090 | 🔴 | **EXPIRY_BATCH outbox channel is terminal-write-only** — COMP-51 writes failure rows with `channel_type=EXPIRY_BATCH`, `status=ERROR` direct (no PUBLISHED→FAILED transition); no platform component is currently confirmed to consume them. COMP-12 Scheduler3 reads only FAILED and PENDING_DEFERRED. | COMP-51, COMP-12 | ADR pending — define consumer or accept as audit-only sink |
| RISK-091 | 🟡 | **Case Expiry Subsystem coordination gap** — COMP-17 (writer) and COMP-51 (reader) share `wdp.case_expiry` with no row-level lock, no version column, no `SELECT FOR UPDATE`. Race on retry-counter UPDATE and on success-path DELETE. | COMP-17, COMP-51 | ADR pending — accept or remediate |

##### 6.1.B Component-specific additions (Phase 3)

###### COMP-01 API Gateway

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-093 | 🔴 | COMP-01 blocking RestTemplate on Netty event-loop with no timeouts on UAMS/CHAS calls — single degraded auth service exhausts all gateway threads | Component file |
| RISK-094 | 🟡 | COMP-01 v-correlation-id always null on UAMS/CHAS calls — cross-service tracing non-functional for all gateway-proxied auth | Subset of RISK-089 |
| RISK-095 | 🟡 | COMP-01 `authExceptions` trim bug — leading-space entries after comma-split silently miss whitelist match | Component file |
| RISK-096 | 🟡 | COMP-01 two unhandled NPE paths → 500 (route attribute null; `AuthorizationList` claim absent) | Component file |

###### COMP-02 UAMS

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-097 | 🔴 | COMP-02 no timeouts on outbound RestTemplate (IdP, S3) | RISK-001 platform pattern |
| RISK-098 | 🟠 | COMP-02 authorization fail-mode on nap-DB outage — every NAP request returns 500 → API Gateway 403 (fail-closed); platform-wide NAP authorization halt on any nap-DB disruption | Architectural side-effect |
| RISK-099 | 🟠 | COMP-02 no idempotency at any UAMS write endpoint — replica-race window | DEC-020 PARTIAL |
| RISK-100 | 🟠 | COMP-02 `updateEntity` NPE — inverted `.get`/`.isPresent` guard returns 500 on missing entity instead of 400 | Component file |
| RISK-101 | 🟡 | COMP-02 topology spread non-functional (`gcp-` prefix drift between pod label and constraint matchLabels) | Component file |
| RISK-102 | 🟡 | COMP-02 unbounded `@Async` fan-out in entity aggregation — heap pressure proportional to tenant size | Component file |
| RISK-103 | 🟡 | COMP-02 `@Async` executor binding non-deterministic — relies on Spring sole-Executor-bean auto-detection | Component file |
| RISK-104 | 🟢 | COMP-02 v-correlation-id not propagated on outbound IdP/S3 calls | Subset of RISK-089 |

###### COMP-03 CHAS

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-105 | 🔴 | COMP-03 `findByCaseNumber` outside try/catch — `DataAccessException` propagates uncaught to default 500 (not GlobalExceptionHandler); API Gateway translates 500 to 403 fail-closed but root cause is invisible | Component file |
| RISK-106 | 🔴 | COMP-03 DB2 MT-path uncaught `DataAccessException` — CH-path equivalent silently swallowed by generic catch | Component file |
| RISK-107 | 🔴 | COMP-03 no JDBC query timeout — neither datasource has socket/query/connection timeout beyond HikariCP pool acquisition default | RISK-001 platform pattern |
| RISK-108 | 🟡 | COMP-03 topology spread label mismatch (`gcp-` prefix) | Same class as RISK-101 |
| RISK-109 | 🟡 | COMP-03 no PodDisruptionBudget | Component file |
| RISK-110 | 🟡 | COMP-03 no CPU limits or requests — Burstable QoS | Component file |
| RISK-111 | 🟡 | COMP-03 no DB connectivity in probes — pod remains traffic-eligible during DB outage; liveness/readiness back onto Spring Actuator health groups with `show-details: never` | Component file |

###### COMP-04 NAPDisputeEventService

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-112 | 🔴 | COMP-04 no HTTP timeout on any RestTemplate (3 instances all default-constructor; one shadowed locally inside `getDisplayCodeDetails`) | RISK-001 platform pattern |
| RISK-113 | 🔴 | COMP-04 blocking synchronous Kafka publish on HTTP thread — `.send.get` with no timeout argument and `@Retryable` on top | DEC-001 deviation |
| RISK-114 | 🔴 | COMP-04 CVV logged in plain text at INFO via `NapEvent.toString` immediately before Kafka publish | RISK-084 cross-link (PCI-DSS) |
| RISK-115 | 🔴 | COMP-04 all endpoints unauthenticated at app level — SecurityConfig whitelist `/**`; auth relies entirely on Ingress / network controls | Component file |
| RISK-116 | 🟡 | COMP-04 `POST /response/document` logs full base64 inbound `file` content via Lombok `toString` at controller entry — Checkmarx sanitizer does not strip the payload | Same family as RISK-114 |
| RISK-117 | 🟡 | COMP-04 retry-config sharing — `${kafka_retry_*}` env vars reused across Kafka publish, CaseManagement (new-event path), FraudSwitch, DisplayCodeService — hidden coupling, tuning Kafka silently retunes 3 REST dependencies | Component file |
| RISK-118 | 🟡 | COMP-04 no application-level inbound idempotency — duplicate POSTs produce duplicate Kafka events; producer idempotence does not span separate inbound HTTP requests | DEC-020 deviation |
| RISK-119 | 🟡 | COMP-04 GUARPAY7 silently maps to TIER5 — downstream cannot distinguish GUARPAY5 from GUARPAY7 by `productSubType` | Component file |
| RISK-120 | 🟢 | COMP-04 `getDisplayCodeDetails` instantiates local `new RestTemplate` that shadows field-level instance | Component file |
| RISK-121 | 🟢 | COMP-04 `@EnableCaching` imported in application class but annotation not applied — Spring caching is not active | Component file |
| RISK-122 | 🟢 | COMP-04 `BeanUtils.copyProperties` silent field-drop risk on future EventRequestDTO ↔ NapEvent name divergence | Component file |

###### COMP-05 NAPDisputeEventProcessor

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-123 | 🟡 | COMP-05 recursive prior-error reprocess on consumer thread can plausibly breach `max.poll.interval.ms` for an ARN with many accumulated FAILED1/FAILED2 rows | Component file |
| RISK-124 | 🟡 | COMP-05 bare RestTemplate with no timeouts on 11 outbound REST calls — at concurrency=1, hung downstream halts all NAP inbound on the pod | RISK-001 platform pattern |
| RISK-125 | 🟡 | COMP-05 MCM Queue multi-result silent ONHOLD with misleading error message — MASTERCARD/MAESTRO disputes silently delayed until `napEventExpiryHour` timeout | Component file |
| RISK-126 | 🟡 | COMP-05 manual reprocess REST + Kafka prior-error scan can race on the same row (no per-record lock) | Mirror of COMP-39 risk class |
| RISK-127 | 🟢 | COMP-05 IDP JWT ThreadLocal not cleared on REST manual-reprocess path — token-leak hazard at concurrency > 1 | Component file |
| RISK-128 | 🟢 | COMP-05 `SEND_BUSINESS_RULES_INFO_TO_KAFKA` is dead infrastructure — silent feature gap, trap for future re-enablement | Component file |

###### COMP-06 NAPDisputeDeclineBatch *(decommission-scope dampening — recorded but not prioritised)*

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-129 | 🟡 | COMP-06 no RestTemplate timeouts | RISK-001 platform pattern |
| RISK-130 | 🟡 | COMP-06 no idempotency guard — `nap.action` not flipped after processing; no idempotency-key on Case Actions POST; duplicate IDCL drafts possible on re-run within `${past_days}` window | DEC-020 deviation |
| RISK-131 | 🟡 | COMP-06 cross-datasource non-atomicity — successful Case Actions POST followed by Spring Batch step-completion write failure on WDP `@BatchDataSource` leaves action created externally but step incomplete | Compounds RISK-130 |

###### COMP-22 DisputeService

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-132 | 🔴 | COMP-22 HikariCP and Tomcat pools at defaults compounded with no client timeouts and no circuit breakers — single slow dependency saturates entire pod with no bulkhead between `/summary` and `/documents` | DEC-014 deviation, blast-radius extension |
| RISK-133 | 🟡 | COMP-22 fully `@Async` SFG SFTP fallback — HTTP 200 returned before SFTP write completes; executor exceptions may be lost; caller has no signal that file landed | Component file |
| RISK-134 | 🟡 | COMP-22 SFG SFTP filename collision — raw `getOriginalFilename` written to `/Outgoing/` with no namespacing | Component file |
| RISK-135 | 🟡 | COMP-22 no idempotency on `POST /documents` — network-retried uploads re-upload, re-transfer ownership, re-publish metadata | DEC-020 deviation |
| RISK-136 | 🟡 | COMP-22 v-correlation-id not propagated to any downstream REST call | Subset of RISK-089 |
| RISK-137 | 🟡 | COMP-22 four required env vars have no defaults — `core_migration_status`, `gcp_async_corepoolsize`, `gcp_async_maxpoolsize`, `gcp_async_queuecapacity` — application fails to start | Component file |
| RISK-138 | 🟢 | COMP-22 401/403 status-code inconsistency — `ForbiddenException` constructed with `HttpStatus.UNAUTHORIZED` but global handler returns 403; constructor's status code is dead | Component file |

###### COMP-25 NotesService

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-139 | 🔴 | COMP-25 mid-batch Kafka orphan window — deterministic split-brain on every multi-note POST that fails partway through publishing; K-1 events on topic with 0 rows in DB after rollback (upgrade from latent-edge-case framing in v1.0) | DEC-001 deviation |
| RISK-140 | 🟡 | COMP-25 `/actuator/prometheus` requires authentication — endpoint exposed but not in Spring Security whitelist; scraper auth required or silent metrics blackout | Component file |
| RISK-141 | 🟡 | COMP-25 `kafka_retry_count` has no default value — service fails to start if env var unset | Component file |
| RISK-142 | 🟡 | COMP-25 `actionSequence` regex-injection latent bug — request value used as regex pattern in `String.matches`; currently mitigated by `@Pattern("^[0-9]*$")` validation, defense-in-depth absent | Component file |
| RISK-143 | 🟢 | COMP-25 Swagger/code drift on NoteType — enum allows 16 values, Swagger `@Schema` declares 14; `NOTE` and `SNOTE` accepted but undocumented | Component file |

###### COMP-26 QuestionnaireService

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-144 | 🔴 | COMP-26 thread-blocking exposure — outbound REST to `case-actions-service` has no connect/read timeout, no retry, no circuit breaker | RISK-001 platform pattern |
| RISK-145 | 🔴 | COMP-26 concurrent-write race on B1 UPSERT and on PUT (no `@Transactional`, no `@Version`, no DB UNIQUE confirmed) — duplicate-row insertion possible | DEC-020 deviation |
| RISK-146 | 🔴 | COMP-26 POST duplicate-row insertion (no app-level pre-existing-row check, no confirmed DB UNIQUE) | DEC-020 deviation |
| RISK-147 | 🟡 | COMP-26 connection-storm exposure — `DriverManagerDataSource` opens a new JDBC connection per JPA call; throughput bounded by Postgres / PgBouncer accept rate | Component file |
| RISK-148 | 🟡 | COMP-26 `HttpMessageNotReadableException` → 500 misleads observability and breaks client error-handling conventions; invalid platform string also returns 500 | Component file |

###### COMP-28 DisplayCodeService

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-149 | 🟠 | COMP-28 permission-shape divergence between POST `/search` (11 flags) and GET `/privileges` (17 flags) — UI portals gated on the 6 extended flags appear unauthorised when rendered from `/search` response | Component file |
| RISK-150 | 🟡 | COMP-28 `UnauthorizedException` (blank JWT `iss` claim) returns HTTP 500 instead of 401 — no dedicated `@ExceptionHandler`; falls through catch-all `RuntimeException` handler | Component file |
| RISK-151 | 🟡 | COMP-28 lazy JWKS resolution — `JwtIssuerAuthenticationManagerResolver` resolves issuer endpoints on first request per issuer, not at startup; cold-start latency anomaly per impacted issuer | Component file |
| RISK-152 | 🟢 | COMP-28 `ValidationMessages.properties` contains messages from a different service (rules-service references) not used by any validator in this service | Component file (hygiene) |

###### COMP-30 UserQueueSkillService

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-153 | 🔴 | COMP-30 POST `/user` user-upsert and auto-enrol are separate implicit transactions — auto-enrol failure leaves orphaned user with no queue assignments | DEC-020 deviation |
| RISK-154 | 🔴 | COMP-30 CHAS infra failure mapped to HTTP 403 — indistinguishable from auth denial; silent service degradation pattern | Component file |
| RISK-155 | 🔴 | COMP-30 `USCaseSearchDaoImpl` swallows DB exceptions during PIN authorization lookup — cascades to misleading 403 | Component file |
| RISK-156 | 🔴 | COMP-30 no XA between `nap` and `wdp` datasources — current paths don't write both schemas atomically, but pattern is one architectural change away from real data inconsistency | Architectural risk |
| RISK-157 | 🔴 | COMP-30 no Resilience4j on IDP calls — portal-access failure cascade for users not yet in local DB during IDP outage | RISK-001 platform pattern |
| RISK-158 | 🟡 | COMP-30 all five REST integrations share one `new RestTemplate` with no timeouts | RISK-001 platform pattern |
| RISK-159 | 🟡 | COMP-30 POST `/lft-report` insert is non-transactional — partial-write windows possible | Component file |
| RISK-160 | 🟡 | COMP-30 no `@UniqueConstraint` on any owned table — POST `/{region}/queue` confirmed race window | DEC-020 deviation |
| RISK-161 | 🟡 | COMP-30 `logback-spring.xml` absent from repo — Logstash integration non-functional unless mounted via ConfigMap/sidecar at runtime | Same class as RISK-050 |
| RISK-162 | 🟢 | COMP-30 `AsyncConfiguration` is dead config — `@EnableAsync` and executor bean created but no `@Async` anywhere | Component file (hygiene) |

###### COMP-32 RulesService

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-163 | 🟠 | COMP-32 DB outage takes down all 14 endpoints simultaneously — no circuit breaker, no socket timeout, no retry, no fallback | RISK-001 platform pattern |
| RISK-164 | 🟠 | COMP-32 `migrationStatus="N"` is an undocumented production migration kill-switch on `/actionrules` — when triggered, every UI action is blocked for the affected case; not in any runbook | Component file |
| RISK-165 | 🟡 | COMP-32 `/actionrules` `AuthorizationList` claim absence returns HTTP 500 NPE in `String.split` instead of HTTP 400 — defeats security-monitoring rules keyed on auth response codes | Component file |
| RISK-166 | 🟡 | COMP-32 `/actionrules` empty-string `AuthorizationList` claim splits to `[""]`, intersection yields empty set, DB query runs with empty IN clause — user receives no permissions silently | Component file |
| RISK-167 | 🟡 | COMP-32 three endpoints (`/notification`, `/business-event`, `/cbk-response`) fall through to HTTP 200 on no-rule-found with three different response shapes while the other 11 throw 404; appears coincidental, not deliberate | Component file |
| RISK-168 | 🟡 | COMP-32 `@Cacheable` key computed from raw method args before in-method blank→`NA` normalisation — duplicate cache entries for null vs `""` inputs that resolve to identical DB queries | Component file |
| RISK-169 | 🟡 | COMP-32 `/documentType` and `/eventRule` controller mappings commented out with full backing code intact — abandon or re-enable? | Component file |

###### COMP-35 EncryptionService

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-170 | 🔴 | COMP-35 single global dependency for all PAN ingestion paths (COMP-07/08/09/11) — outage halts all PAN ingestion | Architectural concentration risk |
| RISK-171 | 🔴 | COMP-35 `rotateDEK` reuse-existing-DEK branch carries Base64 bug — `dek_enc` passed to KMS as raw UTF-8 bytes of Base64 string instead of decoded ciphertext; symptom only manifests on pod restart that finds a recent-enough DEK row to reuse; failure silently swallowed | Component file |
| RISK-172 | 🟡 | COMP-35 decrypt `@Transactional` holds DB connection during KMS call — pool-exhaustion risk under KMS slowdown plus sustained decrypt load | Component file |
| RISK-173 | 🟡 | COMP-35 multi-pod DEK rotation race — no distributed lock | Component file |
| RISK-174 | 🟡 | COMP-35 no DB-level UNIQUE on `wdp.hash_key_store.hpan` verifiable from repo — duplicate-INSERT race across pods | DBA confirmation needed |
| RISK-175 | 🟡 | COMP-35 crypto failures mapped to HTTP 404 — caller error-handling correctness | Component file |
| RISK-176 | 🟡 | COMP-35 DEK startup failure silently swallowed — probes do not check `initialized` flag | Component file |
| RISK-177 | 🟡 | COMP-35 DEK rotation failure silently swallowed — service runs with expired DEK without alert | Component file |
| RISK-178 | 🟢 | COMP-35 `PANHashingServiceImpl` logs last-4 PAN at DEBUG — operational governance dependency | Component file |

###### COMP-39 NAPOutcomeProcessor

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-179 | 🔴 | COMP-39 NAP-DPS authentication relies on out-of-repo network/mesh trust — `napcacrt.jks` is unreferenced orphan; cannot be remediated within component | Infrastructure-layer concern |
| RISK-180 | 🟡 | COMP-39 RestTemplate without timeouts on a single-thread consumer + 8 outbound REST hops | RISK-001 platform pattern |
| RISK-181 | 🟡 | COMP-39 probe path mismatch (`resources.yaml` probes hit `/merchant/gcp/update-consumer/nap/livez` but actuator `additional-path: server:/livez` exposes endpoint at server root) | Component file |
| RISK-182 | 🟡 | COMP-39 manual REST reprocess + Kafka prior-error scan can race on same record (no per-record lock) | Mirror of RISK-126 |
| RISK-183 | 🟡 | COMP-39 `dataRecord` always null in SRV118 (notesLookUp commented out); `checkCRMRAction` commented out — functional gap | Subset of RISK-016 |

###### COMP-40 VisaResponseQuestionnaire

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-184 | 🔴 | COMP-40 `allocarb` `responseType` iterates all `visaResponseIds` with separate RTSI call and document upload per ID, with no transactional bracketing — failure on iteration N leaves 1..N-1 successfully uploaded in DocumentManagementService while iterations N+1..end are never attempted | Component file |
| RISK-185 | 🟡 | COMP-40 IDP token fetched per Kafka message with no caching — `spring-boot-starter-cache` on classpath but no `@EnableCaching`; performance concern at high message volume | Component file |
| RISK-186 | 🟡 | COMP-40 IDP token call positioned BEFORE entry filter — every COMP-19 `AcceptEvent` and every event with null `visaResponseIds` still incurs wasted IDP token RTT before being silently discarded | Compounds RISK-185 |
| RISK-187 | 🟡 | COMP-40 IDP token failure (Step 5) is logged INFO and swallowed by outer Kafka listener catch with offset already committed — no SNOTE written, no error trail | Component file |
| RISK-188 | 🟡 | COMP-40 Kafka listener path performs no MDC enrichment; per-message log correlation depends entirely on OTel agent context | Same gap as COMP-43/17/18/51 |
| RISK-189 | 🟡 | COMP-40 `additionalImagesList` extracted from RTSI but never uploaded to DocumentManagementService — confirmed incomplete work, no TODO, no issue reference | Component file |
| RISK-190 | 🟢 | COMP-40 `${app.name}` referenced in `application.yml` for `management.metrics.tags.application` but never defined — Prometheus tag effectively unset | Component file |
| RISK-191 | 🟢 | COMP-40 `KafkaConsumer.java` log label reads `key-MerchantId` but binds the `caseNumber` variable — observability noise | Component file |

###### COMP-42 BENConsumer

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-192 | 🔴 | COMP-42 PUBLISHED-orphan crash window Step 4 → Step 13 — same class as RISK-040 (COMP-41); Scheduler3 invisibility, no automatic recovery | Extends RISK-040 |
| RISK-193 | 🟡 | COMP-42 all six outbound dependencies have no read/connect timeout configured | RISK-001 platform pattern |
| RISK-194 | 🟢 | COMP-42 predecessor lookup loads ALL channel_types for caseNumber, filters to BEN_EVENTS in memory — unbounded per-case query, no pagination, no LIMIT | Component file |

###### COMP-49 WDP Portal

*(Added 2026-05-06 in post-baseline reconciliation against COMP-49 source-verification.)*

| Risk ID | Severity | Risk | Reference |
|---|---|---|---|
| RISK-195 | 🟠 | COMP-49 OPS super-user UI bypass — `WdpUtilService.canUserPerformDisputeAction()` returns `{canPerform: true}` unconditionally for any OPS_USER. Combined with DEC-018 (no server-side RBAC on COMP-24), there is no gate at either UI or backend layer. Any authenticated OPS user can take any dispute action — including the eleven More Actions — regardless of case state, queue assignment, or business-rule eligibility. **Highest-impact finding from COMP-49 audit.** | DEC-018 — amplified; ADR-CAND-NEW-A |
| RISK-196 | 🟠 | COMP-49 `console.log` / `console.error` / `console.debug` statements (~120+) are not stripped from production builds, including in auth-handling paths. PII / token / case-data leakage to browser DevTools console. | ADR-CAND-NEW-B |
| RISK-197 | 🟠 | COMP-49 UI permissions fetched from `/display-code/privileges` once at app bootstrap and cached in `WdpSharedService` for the full session. Mid-session permission revocation is not honoured until logout. | ADR-CAND-NEW-C |
| RISK-198 | 🟠 | COMP-49 accepts full PAN at three input forms across both modes: Filter Disputes (both), Filter Fax Dispute (Ops), Update Transaction Search (Ops). Encrypted in transit; decrypted PAN held in AG Grid memory only — no client-side persistence — but the input-layer surface is in PCI scope. | DEC-019 — PARTIAL |
| RISK-199 | 🟡 | COMP-49 — five previously undocumented More Actions surfaced in source: Reverse, Issuer Reversal, Issuer Accept, Convert to Partial DTP, Update Transaction. Each carries its own form, payload, and backend effect. Audit and risk assessment of the previously undocumented actions required. *Note: COMP-49 source file rates this 🔴 HIGH; rated 🟡 MEDIUM here per change-log specification — gap is medium until per-action audit completes.* | ADR-CAND-NEW-D |
| RISK-200 | 🟢 | COMP-49 `v-correlation-id` header is timestamp-derived UUID, not cryptographically random. Predictable. Used for trace correlation, not security. | Component file |

##### 6.1.C Pattern Extensions (existing rows extended with newly-audited confirmations)

The following existing risk rows had their **affected-components list extended** during this reconciliation pass. No new RISK IDs assigned — the underlying finding is unchanged, only the scope.

| Existing Risk | Extension |
|---|---|
| **RISK-001** No circuit breakers / no timeouts | Extended to include COMP-01, 02, 03, 04, 05, 06, 22, 25, 26, 30, 32, 35, 39, 42 explicitly. All 38 source-verified components confirm. |
| **RISK-010** UAMS wrong transaction manager | **Promoted from 🟠 to 🔴.** Scope expanded from 1 method (`saveChildWithMerchant`) to 7 methods across 4 NAP tables. COMP-30 added as second offender. |
| **RISK-012** validateOrgId commented out | Extended with v1.1 finding that the method body is itself absent from `RequestValidator`. |
| **RISK-013** Polling batch replica constraint | Extended to include COMP-06 (NAPDisputeDeclineBatch) and COMP-51 (CaseExpiryProcessor). |
| **RISK-024** No K8s probes | Extended to include COMP-39 (no startup probe), COMP-42 (no probes at all), COMP-51 (no probes at all). |
| **RISK-025** Empty `CommonErrorHandler` silent swallow | Extended to include COMP-05, COMP-40, COMP-42. Now 10 components confirmed. |
| **RISK-040** PUBLISHED-orphan paths | Extended to COMP-42 BEN_EVENTS via RISK-192 (same class as the COMP-41 paths). |
| **RISK-050** Empty Logstash secret | Extended pattern to COMP-30 `logback-spring.xml` absent from repo (RISK-161). |
| **RISK-076** `spring-boot-devtools` in production | Extended to include COMP-30 confirmed. |
| **RISK-077** Outbox rows carry no Kafka-coordinate columns | Extended pattern: now five outbox writers (COMP-17, 41, 42, 43, 51) all confirm no Kafka-coordinate columns. |

#### Reconciliation summary (Phase 3)

- **Rows added:** 110 new RISK rows in Phase 3 main pass (RISK-083 through RISK-091 platform-wide; RISK-093 through RISK-194 component-specific). Plus 6 additional rows in 2026-05-06 post-baseline reconciliation (RISK-195 through RISK-200, all COMP-49 WDP Portal). **Total: 116.**
- **Rows promoted:** RISK-010 promoted from 🟠 to 🔴 to reflect DEC-021 scope expansion.
- **Rows extended:** RISK-001, 012, 013, 024, 025, 040, 050, 076, 077 (affected-components list).
- **Rows withdrawn:** None in this pass.
- **New ADR candidates surfaced (most urgent):**
  - RISK-084 — CVV-at-rest exception decision (PCI-DSS 3.2.1)
  - RISK-085 — shared-error-table consumption rules (COMP-05/COMP-39 uniform decision)
  - RISK-086 — DEC-021 wrong-TM scope expansion (now 7 methods, 2 offenders)
  - RISK-090 — EXPIRY_BATCH terminal-write-only consumer decision
  - RISK-091 — Case Expiry Subsystem coordination (COMP-17 ↔ COMP-51 shared write)

---

### 6.2 Critical Risk Details (RISK-001 to RISK-004 — Phase 1)

(Unchanged from v2.1 — see WDP-NFRS.md narrative.)

---

### 6.3 Critical Risk Details (RISK-024 onwards — Phases 2 and 3)

For brevity, Phase 2 critical risks were narrated in v2.1 (RISK-024, RISK-025, RISK-028, RISK-035, RISK-038). The following Phase 3 critical risks warrant narrative emphasis.

---

**RISK-084: CVV at rest — PCI-DSS 3.2.1 prohibition**
**Severity:** 🔴 CRITICAL | **Reference:** ADR pending — DEC-019-class

PCI-DSS 3.2.1 Requirement 3.2 prohibits storing the CVV (CVV2/CVC2/CID) after authorisation under any circumstance. Two confirmed paths in WDP currently violate this:

1. **COMP-05 — CVV at rest in error table.** `NAP.DISPUTE_EVENT_CONSUMER_ERROR` persists CVV in **two** places per error row: the `C_CVV` column directly mapped on the entity, AND inside `C_KAFKA_EVENT` raw JSON which retains the inbound `cvv` field. Every NAP dispute event that fails primary processing produces a row with CVV on disk.
2. **COMP-04 — CVV in logs.** `NapEvent.toString` is generated by Lombok and surfaces the `cvv` field. The service logs the outbound event at INFO immediately before Kafka publish. Every published NAP event produces a log line containing CVV. Checkmarx sanitizer is invoked but does not strip the field.

Together these constitute a CVV-on-disk path: published events are logged with CVV; failed events persist CVV in error tables; both feed downstream log aggregation pipelines that themselves persist for audit.

**Architect decision required:**
- Remediate (clear `C_CVV` column on insert in COMP-05; redact `C_KAFKA_EVENT` JSON; remove `toString` from logging surface in COMP-04; enforce log scrubbing on Logstash side); OR
- Document approved exception with compensating controls.

This is a **NEW MATERIAL DEFICIENCY** under PCI-DSS 3.2.1 active scope.

---

**RISK-085: Cross-component shared-error-table consumption hazard**
**Severity:** 🔴 CRITICAL | **Reference:** ADR pending — uniform decision required

`NAP.DISPUTE_EVENT_CONSUMER_ERROR` is now confirmed to have **four writers**: COMP-05 (primary), COMP-23 (NAP create path blind-merge), COMP-24 (NAP conditional outbox), COMP-39 (manual reprocess + outbound NAP write). Discriminator is `C_ACQ_PLATFORM` (`"NAP"` constant) plus `C_EVENT_TYPE` (`OUT_SRV118`, `OUT_SRV117` for COMP-39).

**The hazard:** both COMP-05 and COMP-39 prior-error scans query this table without filtering on `C_EVENT_TYPE`. COMP-05's recursive prior-error scan (looking for FAILED1/FAILED2 rows for the same `caseNumber`) can pick up rows written by COMP-39 (or COMP-23 / COMP-24) and reprocess them through COMP-05's inbound NAP pipeline. COMP-39's prior-error scan has the same defect in the outbound direction.

**Architect decision required:** Is the missing `C_EVENT_TYPE` filter intentional (cross-pipeline recovery) or a defect (event-type mixing)? Decision must be uniform — COMP-05 and COMP-39 cannot adopt different policies on the same table.

---

**RISK-090: EXPIRY_BATCH outbox channel is terminal-write-only**
**Severity:** 🔴 CRITICAL | **Reference:** ADR pending

COMP-51 CaseExpiryProcessor writes failure-capture rows to `wdp.outgoing_event_outbox` with:
- `channel_type = EXPIRY_BATCH` (distinct from COMP-17's `EXPIRY_EVENTS`)
- `created_by = WCSEEXPB` (distinct from COMP-17's `WCSEEXPC`)
- `status = ERROR` direct (no PUBLISHED → FAILED transition)

COMP-12 Scheduler3 reads only `status IN (FAILED, PENDING_DEFERRED)` — so EXPIRY_BATCH ERROR rows are **terminal-write-only with no recovery path**. They are an audit sink that no automated process drives.

**Architect decision required:**
- Define an explicit consumer for EXPIRY_BATCH ERROR rows (e.g. extend Scheduler3 to read this channel; add manual operator runbook); OR
- Formally accept the rows as audit-only sink with no recovery semantics.

---

**RISK-093: COMP-01 blocking RestTemplate on Netty event-loop**
**Severity:** 🔴 CRITICAL | **Reference:** Component file

API Gateway is the platform's externally-visible entry point for merchants and internal callers. It uses Spring Cloud Gateway on Netty, which is a reactive non-blocking framework — but the gateway makes blocking `RestTemplate` calls to UAMS and CHAS for case-level authorization. Neither RestTemplate has connect/read timeouts. A single degraded auth service (UAMS or CHAS) holds Netty event-loop threads indefinitely.

**Blast radius:** the entire gateway. Once event-loop threads are exhausted, no merchant or internal traffic is processed regardless of which downstream service is degraded.

**Architect decision required:** migrate UAMS / CHAS calls to `WebClient` (reactive) with explicit timeouts; or accept the latency floor and document operational mitigation.

---

**RISK-115: COMP-04 all endpoints unauthenticated at app level**
**Severity:** 🔴 CRITICAL | **Reference:** Component file

COMP-04 NAPDisputeEventService SecurityConfig contains a `permitAll` whitelist with `/**` pattern. No application-level JWT validation, no application-level role check. Authorization for all NAP dispute event ingestion is enforced **entirely at the Ingress / network layer**.

The dependency on infrastructure-layer auth has two failure modes:
1. Misconfigured Ingress allows unauthenticated traffic to reach the service.
2. Internal cluster traffic bypasses Ingress entirely (e.g. service mesh mis-route, debugging port-forward).

**Architect decision required:** add Spring Security JWT validation; or document that infrastructure-layer auth is sufficient with operational mitigations (network policy, service-mesh mTLS).

---

**RISK-170: COMP-35 single global dependency for all PAN ingestion paths**
**Severity:** 🔴 CRITICAL | **Reference:** Architectural concentration risk

EncryptionService (COMP-35) is the sole encryption authority for all PAN ingestion in WDP. COMP-07, COMP-08, COMP-09, COMP-11 all call COMP-35 for PAN encryption before persisting any chargeback or case record. COMP-35 outage → all four ingestion paths halt.

Compounding: COMP-35 has no circuit breaker, no rate limiter, no bulkhead between concurrent decrypt/encrypt callers. KMS slowdown propagates directly to all PAN-handling components.

**Architect decision required:** define HA posture for COMP-35 (replica count, KMS quota allocation, regional failover); accept concentration risk with documented degraded-mode procedure; or evaluate in-process encryption sidecar pattern as alternative.

---

### 6.4 High Risk Details (RISK-005 to RISK-012 — Phase 1)

(Unchanged from v2.1.)

---

### 6.5 High Risk Details (RISK-044 onwards — Phase 2)

(Unchanged from v2.1 — see RISK-040 narrative and RISK-046 narrative.)

---

### 6.6 High Risk Details (Phase 3)

The Phase 3 HIGH-severity risks are documented in the summary table above. The following warrant narrative emphasis.

---

**RISK-098: COMP-02 authorization fail-mode on nap-DB outage — platform-wide NAP halt**
**Severity:** 🟠 HIGH | **Reference:** Architectural side-effect

UAMS (COMP-02) is the sole authorization authority for NAP platform requests. Every NAP request flows through API Gateway → UAMS `/authorize`. Any DB outage on the `nap` schema causes UAMS to return HTTP 500. API Gateway (COMP-01) is configured fail-closed: 500 from UAMS → 403 to caller.

**Net effect:** any nap-DB disruption — even brief — halts ALL NAP authorization platform-wide. There is no caching layer, no degraded-mode policy, no fallback to last-known-good.

**Mitigations to consider:** UAMS authorization cache with TTL (e.g. ElastiCache); fail-open policy for known-low-risk endpoints during DB outage; circuit breaker to fail fast and surface 503 instead of 403.

---

**RISK-149: COMP-28 permission-shape divergence**
**Severity:** 🟠 MEDIUM-HIGH | **Reference:** Component file

COMP-28 DisplayCodeService has two repository projections on `wdp.dispute_static_tabs_rules` that return **different column counts** under the same nominal `UserPermission` envelope:

- `findByRoles` (POST `/search`) returns 11 Y/N flag columns
- `getPermissions` (GET `/privileges`) returns 17 Y/N flag columns

The 6-column delta (`fax_match`, `fax_report`, `trans_detail`, `auth_detail`, `settle_detail`, `dispute_history`) is the source of permission-shape divergence. UI portals gated on the 6 extended flags appear unauthorised when the page is rendered from `/search` response data — even when the caller's roles authorise them.

**Architect decision required:** is the minimised projection on `/search` intentional (e.g. for performance or call-site appropriateness) or accidental drift from the original 17-flag schema?

---

**RISK-195: COMP-49 OPS super-user UI bypass amplifies DEC-018**
**Severity:** 🟠 HIGH | **Reference:** DEC-018 — amplified; ADR-CAND-NEW-A

`WdpUtilService.canUserPerformDisputeAction()` (`wdp.util.service.ts:44-46`) returns `{canPerform: true}` unconditionally for any OPS_USER, with no inspection of action type, case state, queue assignment, or per-action permissions. Combined with DEC-018 (CaseActionService COMP-24 has no server-side RBAC), there is no enforcement gate at either UI or backend layer for OPS_USER. Any authenticated OPS user can invoke any dispute action — including the eleven More Actions — regardless of business state.

This is the highest-impact finding from the COMP-49 source-verification audit. Whether the unconditional `true` is intentional (treating OPS as a trusted superuser) or a defect to remediate is captured as **ADR-CAND-NEW-A** and is flagged as top-priority for architect resolution.

---

### 6.7 Medium and Low Risk Details (Phases 2 and 3)

For completeness, all Phase 2 and Phase 3 MEDIUM and LOW risks are recorded with sufficient detail in the summary table at Section 6.1. Component-specific narrative is captured in the relevant component file (WDP-COMP-NN-*.md) and in WDP-HANDOVER.md "Component-Specific Source-Verified Findings".

---

### 6.8 Component-to-Risk Index
Navigation aid for finding all risks affecting a given component.

| Component | Phase 1 risks | Phase 2 risks | Phase 3 risks |
|-----------|---------------|---------------|---------------|
| COMP-01 API Gateway | | | RISK-083 (extends), RISK-089, **RISK-093, 094, 095, 096** |
| COMP-02 UAMS | RISK-010 (PROMOTED 🔴) | | **RISK-097, 098, 099, 100, 101, 102, 103, 104** |
| COMP-03 CHAS | RISK-012 | | RISK-083, RISK-086 (related), **RISK-105, 106, 107, 108, 109, 110, 111** |
| COMP-04 NAPDisputeEventService | | | RISK-084, RISK-001 (extends), **RISK-112, 113, 114, 115, 116, 117, 118, 119, 120, 121, 122** |
| COMP-05 NAPDisputeEventProcessor | | | RISK-083, RISK-084, RISK-085, RISK-025 (extends), **RISK-123, 124, 125, 126, 127, 128** |
| COMP-06 NAPDisputeDeclineBatch | | | RISK-013 (extends), RISK-024 (extends), **RISK-129, 130, 131** |
| COMP-07 VisaDisputeBatch | RISK-013, 014 | RISK-024 | |
| COMP-08 FirstChargebackBatch | RISK-013, 014 | RISK-024, 052, 061, 062 | RISK-083 |
| COMP-09 CaseFillingBatch | RISK-013 | RISK-024, 053, 063 | RISK-083 |
| COMP-11 FileProcessor | | RISK-024, 054, 064, 065, 073, 074 | |
| COMP-12 InboundDisputeEventScheduler | RISK-015 | RISK-024, 038, 077 | RISK-083, RISK-088 |
| COMP-13 FileAcknowledgementProcessor | | RISK-056 | RISK-088 |
| COMP-14 CaseCreationConsumer | | RISK-024, 025, 072, 082 | |
| COMP-15 EvidenceConsumer | | RISK-002, 025, 029, 030, 081 | |
| COMP-16 BusinessRulesProcessor | RISK-007, 008, 011, 020, 022 | RISK-025 | |
| COMP-17 CaseExpiryUpdateConsumer | | RISK-024, 025, 031, 045, 077, 079 | RISK-089, RISK-091 |
| COMP-18 NotificationOrchestrator | RISK-015 | RISK-025, 077 | |
| COMP-19 AcceptService | RISK-018 | RISK-028, 044 | |
| COMP-20 ContestService | RISK-018 | RISK-044 (likely shared) | |
| COMP-21 ChargebackService | | RISK-026, 027, 049, 050, 051, 070, 071 | |
| COMP-22 DisputeService | RISK-021 | | RISK-089, **RISK-132, 133, 134, 135, 136, 137, 138** |
| COMP-23 CaseManagementService | RISK-004, 006 | RISK-032, 033, 067, 068, 069, 076 | |
| COMP-24 CaseActionService | RISK-005 | RISK-039, 046, 047, 048, 075, 076 | |
| COMP-25 NotesService | | | RISK-083, **RISK-139, 140, 141, 142, 143** |
| COMP-26 QuestionnaireService | | | RISK-083, **RISK-144, 145, 146, 147, 148** |
| COMP-27 CaseSearchService | | RISK-041, 042, 043, 078 | |
| COMP-28 DisplayCodeService | RISK-023 | | RISK-083, **RISK-149, 150, 151, 152** |
| COMP-30 UserQueueSkillService | | | RISK-010 (extended), RISK-076 (extended), **RISK-153, 154, 155, 156, 157, 158, 159, 160, 161, 162** |
| COMP-32 RulesService | | | **RISK-163, 164, 165, 166, 167, 168, 169** |
| COMP-34 MerchantTransactionService | RISK-019 | | RISK-083 |
| COMP-35 EncryptionService | | | **RISK-170, 171, 172, 173, 174, 175, 176, 177, 178** |
| COMP-37 DocumentManagementService | | RISK-034, 066 | |
| COMP-39 NAPOutcomeProcessor | RISK-016 | RISK-025 | RISK-085, RISK-024 (extends), **RISK-179, 180, 181, 182, 183** |
| COMP-40 VisaResponseQuestionnaire | RISK-017 | | RISK-083, RISK-025 (extends), **RISK-184, 185, 186, 187, 188, 189, 190, 191** |
| COMP-41 ThirdPartyNotificationConsumer | | RISK-024, 025, 040, 055, 080 | |
| COMP-42 BENConsumer | | RISK-025 | RISK-024 (extends), RISK-040 (extends), **RISK-192, 193, 194** |
| COMP-43 CoreNotificationConsumer | | RISK-024, 025, 035, 036, 037, 057, 058, 059, 060, 077 | |
| COMP-49 WDP Portal (Merchant + Ops modes) | | | **RISK-195, 196, 197, 198, 199, 200** |
| COMP-51 CaseExpiryProcessor (NEW) | | | RISK-013 (extends), RISK-024 (extends), RISK-077 (extends), RISK-089, RISK-090, RISK-091 |

---

*This document covers non-functional requirements and confirmed platform risks.*
*NFR targets for unaddressed risks to be set by product team.*
*Implementation details, database schemas, and deployment specifications are maintained in individual component files (WDP-COMP-[NN]-*.md).*
