# WDP-ARCHITECTURE.md
**Worldpay Dispute Platform — Architecture Reference**
*Version: 2.2 | Reconciled: 2026-04-30*
*Source: v2.1 (2026-04-25) + 2026-04-28/29/30 source-verification reconciliation against 18 components*

*v2.2 reconciliation scope: minor topology-level clarifications only. The 18 component-file audits surfaced primarily component-level findings recorded in WDP-NFRS.md v2.2 (RISK-083 to RISK-194) and WDP-DECISIONS.md v2.2 (ADR-CAND-023 to ADR-CAND-059). Architecture-level changes are limited to: (a) component count update 50 → 51 (COMP-51 CaseExpiryProcessor added; grouped with COMP-17 under "Case Expiry Subsystem"), (b) UAMS 4-path summary added to §3 / §4.5, (c) CHAS endpoint count corrected to 9 and scope confirmed as PIN+CORE (not PIN-only), (d) §3.4 Authentication & Authorization extended with COMP-01 CORE/VAP/LATAM gateway gap, COMP-04 unauthenticated `/**` whitelist, and COMP-22 internal-firm enforcement detail, (e) §4.7 / §11.1 DEK rotation interval correction (days, not 6 hours — DEC-008 correction), (f) §4.8 Core Data Stores datasource-pattern note extended with COMP-22, COMP-26, COMP-32, COMP-02, (g) §10.1 PAN Encryption extended with NEW MATERIAL DEFICIENCY: CVV-at-rest in COMP-04 + COMP-05, (h) §10.2 Resilience pattern extended with CommonErrorHandler list to 10 components and minReadySeconds platform-wide finding, (i) §10.4 Compliance refreshed with PCI-DSS 3.2.1 deficiency callout, (j) §11.1 ElastiCache rewritten — TokenService is now confirmed read-only on Redis, (k) §12 Component Status Registry refreshed across multiple sub-tables, (l) Open Discussion Points: 4 resolved, 8 new architecture-level entries.*

---

## Table of Contents

1. [Purpose & Context](#1-purpose--context)
2. [System Context](#2-system-context)
3. [UI & Access Layer](#3-ui--access-layer)
4. [WDP Core Services](#4-wdp-core-services)
5. [Inbound Processing](#5-inbound-processing)
6. [Kafka Event Bus](#6-kafka-event-bus)
7. [Event Consumers](#7-event-consumers)
8. [Outbound Processing](#8-outbound-processing)
9. [Acquiring Platform Integration](#9-acquiring-platform-integration)
10. [Cross-Cutting Concerns](#10-cross-cutting-concerns)
11. [Deployment Context](#11-deployment-context)
12. [Component Status Registry](#12-component-status-registry)

---

## Knowledge Base Navigation

This document covers platform-level topology, principles, and deployment context. It is the starting point — not the complete picture.

For deeper detail, use the documents below. Claude searches all project files simultaneously, so these cross-references help orient every response toward the right source of truth.

| Question type | Go to |
|---------------|-------|
| What does component X do? | WDP-COMP-INDEX.md → WDP-COMP-[NN]-*.md |
| What are the REST endpoints for component X? | WDP-COMP-[NN]-*.md Block A |
| Which Kafka topics does component X produce or consume? | WDP-COMP-[NN]-*.md Block B/C + WDP-KAFKA.md |
| Which database tables does component X own? | WDP-COMP-[NN]-*.md + WDP-DB.md |
| How does a dispute flow end to end? | WDP-FLOW-INDEX.md → WDP-FLOW-*.md |
| What are the architecture decisions and tradeoffs? | WDP-DECISIONS.md |
| What does WDP integrate with externally? | WDP-INTEGRATIONS.md |
| What are the performance and resilience constraints? | WDP-NFRS.md |
| What is the current work position and session context? | WDP-HANDOVER.md |

**Document status as of 2026-04-30:**
- WDP-ARCHITECTURE.md — ✅ Current v2.2 (this document)
- WDP-COMP-INDEX.md — ✅ Current v2.2 (51 components, 38 source-verified)
- WDP-KAFKA.md — ✅ Current v2.2 (5-channel outbox, hidden coupling, operational footgun)
- WDP-DB.md — ✅ Current v2.2 (5 outbox writers, CVV PCI-DSS finding documented, 13 new rows)
- WDP-FLOW-INDEX.md — ✅ Current v1.1 (Case Expiry Subsystem captured; Per-Flow Authoring Notes added)
- WDP-COMP-[NN]-*.md — 📝 38 DRAFT 🔍 (source-verified between 2026-04-18 and 2026-04-30; architect confirmation pending), 4 audit-pending (COMP-29, 31, 34, 38), 1 PENDING (COMP-10), 6 NOT STARTED, 2 UI separate
- WDP-DECISIONS.md — ✅ Current v2.2 (DEC-008 DEK rotation corrected; DEC-019 CVV exception; DEC-021 scope expanded; 37 new candidate ADRs)
- WDP-INTEGRATIONS.md — ✅ Current v2.2 (AWS Secrets Manager added §6.9; UAMS external surface; SFG SFTP NAP fallback; NAP-DPS auth gap)
- WDP-NFRS.md — ✅ Current v2.2 (110 new RISK rows added in Phase 3, RISK-083 to RISK-194; RISK-010 promoted 🟠→🔴)
- WDP-HANDOVER.md — ✅ Current v3.2 (reconciled 2026-04-30)
- WDP-CHANGE-LOG.md — Active append-only log; 38 entries reconciled total (20 in 2026-04-25 pass + 18 in 2026-04-30 pass)

---

## 1. Purpose & Context

[Content unchanged from v2.1.]

---

## 2. System Context

[Content unchanged from v2.1 — Sections 2.1, 2.2, 2.3, 2.4 retained without change. The High Level Architecture Overview Mermaid diagram in §2.2 represents the full topology as of v2.1; the v2.2 addition of COMP-51 is captured in §7.2 / §12.7 / new "Case Expiry Subsystem" sub-grouping rather than altering the headline diagram.]

---

## 3. UI & Access Layer

[Sections 3.1, 3.2 unchanged from v2.1.]

### 3.3 API Gateway

[Content unchanged from v2.1.]

⚠️ **(2026-04-30) Confirmed gap — CORE/VAP/LATAM gateway-level authorization absent.** Source verification of COMP-01 confirmed **no CORE platform value exists in API Gateway source code**. CORE, VAP, and LATAM platform requests receive **no role-level or case-level authorization at the gateway layer**. NAP and PIN requests are routed through the case-level authorization filter (UAMS / CHAS); CORE/VAP/LATAM bypass it entirely. Architect decision required — see WDP-DECISIONS.md ADR-CAND-029.

⚠️ **(2026-04-30) Resilience gap — blocking RestTemplate on Netty event-loop.** API Gateway uses Spring Cloud Gateway on Netty (a reactive non-blocking framework), but `CaseNumberFilter` invokes UAMS and CHAS using **blocking `RestTemplate` on Netty event-loop threads with no timeout**. A single degraded auth service can exhaust all gateway threads. Blast radius: the entire gateway. See WDP-NFRS.md RISK-093 and WDP-DECISIONS.md ADR-CAND-028.

⚠️ **(2026-04-30) Listen port confirmed `8082`.** Prior uncertainty about port 9082 was a documentation error.

### 3.4 Authentication & Authorization ⚠️ Refined 2026-04-29 / 30

[Existing v2.1 content preserved.]

⚠️ **(2026-04-29) UAMS — 4-path summary (replaces v2.1 "detailed behaviour to be confirmed"):**
1. **NAP `POST /authorize`** — case-level authorization for NAP requests (3-layer model: consumer authorization → entity-class gate → entity-value lookup).
2. **Entity CRUD** — six write endpoint families across `nap_parent_entity`, `nap_child_entity`, `nap_merchant`, `nap_entity_rel`. Application-level SELECT-then-INSERT only; no DB UNIQUE visible. ⚠️ **DEC-021 wrong-TM scope expansion (RISK-010 promoted 🟠→🔴):** 7 methods write to NAP-schema tables under `wdpTransactionManager` instead of `napTransactionManager`. See WDP-DECISIONS.md DEC-021.
3. **SunGard IdP user-lifecycle proxy** — two firm-routed instances (`Merchant_Fraud_Disputes` → MFD; else → standard `US_Merchant`). See WDP-INTEGRATIONS.md §6.1.
4. **AWS S3 bulk onboarding** — direct write to `${wdp_entity_file}` bucket, key prefix `RECEIVED/{ENV}/`, hardcoded `eu-west-2` region. ⚠️ **No downstream parser identified** — open question. See WDP-INTEGRATIONS.md §7.2.

⚠️ **(2026-04-29) CHAS — endpoint count corrected to 9** (v1.0/v2.0 documents stated 6). V2 controller adds three endpoints: `POST /{platform}/v2/merchant/entitytype` (paginated), `GET /{platform}/v2/merchant/orgentity/{orgId}`, `GET /{platform}/v2/merchant/defaultentity/{orgId}`. **Scope confirmed PIN+CORE** (`RequestValidator.validatePlatform` accepts both, not PIN-only as v1.0 stated). The data API surface serves both PIN and CORE platform consumers.

⚠️ **(2026-04-29) `validateOrgId()` v1.1 finding:** Already documented as commented-out in v2.1. Source verification adds that **the method body is itself absent** from `RequestValidator` — remediation cannot be done by uncomment alone; reimplementation required. See RISK-012, ADR-CAND-029-class.

⚠️ **(2026-04-29) Internal firm bypass scope:** Bypass (`iss` contains `us_worldpay_fis_int`) applies to **BOTH** `/authorize` AND `/entity-authorize` — both share `AuthorizationServiceImpl.authorizeEntity`. Plus partial-bypass on `POST /{platform}/merchant/entitytype` (V1 and V2): internal callers skip JWT `iqentities` extraction and use `orgId` from request body directly.

⚠️ **(2026-04-29) NEW: COMP-04 unauthenticated SecurityConfig.** All COMP-04 NAPDisputeEventService endpoints are unauthenticated at app level — SecurityConfig whitelist is `/**`. Auth relies entirely on Ingress / network controls. See WDP-NFRS.md RISK-115 and WDP-DECISIONS.md ADR-CAND-031.

⚠️ **(2026-04-28) NEW: COMP-22 internal-firm enforcement is application-layer (not Spring Security).** `contains` check on JWT `iss` claim against literal `us_worldpay_fis_int`. `ForbiddenException` constructor uses `HttpStatus.UNAUTHORIZED` but global handler returns 403 — constructor's status code is dead. See RISK-138.

⚠️ **(2026-04-30) Confirmed pattern: 3-layer authorization model.** Both COMP-02 UAMS and COMP-03 CHAS implement the same shape: (1) Spring Security JWT validation, (2) internal-firm bypass on `iss` substring `us_worldpay_fis_int`, (3) entity-scope check against the platform-specific relationship table. Same constants, same shape, two implementations. Candidate to formalise as a platform pattern — see ADR-CAND-038.

---

## 4. WDP Core Services

### 4.1 Case Action Services

[Content unchanged from v2.1.]

### 4.2 Search & Display Services

[Content unchanged from v2.1.]

### 4.3 Queue & Workflow Services

[Content unchanged from v2.1.]

### 4.4 Rules & Configuration Services

[Content unchanged from v2.1.]

⚠️ **(2026-04-28) RulesService (COMP-32) — confirmed pure read-only.** Zero writes, zero Kafka, zero PAN, zero outbound REST/HTTP, zero `@Transactional` mutations. Hosts 14 active REST endpoints + 2 controller-disabled endpoints with full backing code intact (`/documentType`, `/eventRule` — abandon-or-re-enable decision pending; ADR-CAND-050). 14 active endpoints map 1:1 to 14 distinct `@Cacheable` cache names backed by Spring's default `ConcurrentMapCacheManager`. ⚠️ **Production migration kill-switch (RISK-164 / ADR-CAND-049):** `/actionrules` `migrationStatus="N"` is an **undocumented production migration kill-switch** — when triggered, every UI action is blocked for the affected case. Not in any runbook.

### 4.5 Organisation & User Management Services ⚠️ Refined 2026-04-29

**OrgManagementService** manages organisational hierarchies within WDP. Component documentation deferred — GitHub repository not found.

**CoreHierarchyAuthorizationService (COMP-03 CHAS)** enforces authorization based on the organisational hierarchy. **9 REST endpoints** across two controllers (corrected from v1.0's 6). `RequestValidator.validatePlatform` accepts **both PIN and CORE** — the data API surface serves both PIN- and CORE-platform consumers. Internal firm bypass (`iss` contains `us_worldpay_fis_int`) applies to both `/authorize` and `/entity-authorize`; partial-bypass on `POST /{platform}/merchant/entitytype` (V1 and V2). See §3.4 for full scope and known gaps (`validateOrgId` method body absent).

**UserAccessManagementService (COMP-02 UAMS)** — 4-path summary added in §3.4. Authorization fail-mode on nap-DB outage is platform-wide NAP halt: every NAP request returns 500 → API Gateway 403 fail-closed (RISK-098).

⚠️ **(2026-04-29) UserQueueSkillService (COMP-30) — DEC-021 second offender.** Service-level `@Transactional` on `createQueue`/`updateQueue` binds to `@Primary usTransactionManager` because `jakarta.transaction.Transactional` default-bean-selection silently picks `@Primary`. UK writes to `nap.queues`, `nap.queue_criterion`, `nap.user_queue` are NOT covered by the outer TX. Same root cause class as COMP-02 — see WDP-DECISIONS.md DEC-021 + ADR-CAND-033 (multi-datasource binding contract).

### 4.6 Platform Integration Services

[Content unchanged from v2.1.]

### 4.7 Supporting Services ⚠️ Corrected 2026-04-29

**EncryptionService (COMP-35)** is the sole component authorised to handle plaintext PAN data in WDP. It encrypts PANs at the point of ingestion and provides decryption only to authorised callers. No other service stores or processes raw cardholder data. Full details are covered in Section 10.1.

⚠️ **(2026-04-29) DEK rotation interval correction:** Prior platform documentation stated a "6-hour DEK cache." Source verification confirms this is **incorrect**. The actual rotation interval is **days**, configured via `${dek_rotation_interval_days}`. See WDP-DECISIONS.md DEC-008 correction.

⚠️ **(2026-04-29) EncryptionService is a single global dependency for all PAN ingestion (RISK-170).** COMP-07 / COMP-08 / COMP-09 / COMP-11 all call EncryptionService for PAN encrypt; COMP-43 for HPAN decrypt. Outage halts all four ingestion paths. HA posture is an open architectural question.

**TokenService (COMP-36)** is the centralised JWT token management service. ⚠️ **(2026-04-29) Source-verified read-only on Redis** — performs HGET only; zero write operations exist anywhere in the wdp-idp-token-service repository. The `wdpinternalidptoken:token` Redis hash is populated by an UNKNOWN EXTERNAL COMPONENT not present in any audited WDP repo. External writer identity is an open question. Two-layer cache: AWS ElastiCache Redis (primary) + Spring in-memory OAuth2 store (secondary). Falls through to IDP `client_credentials` grant only on full cache miss.

**APILogService** provides API-level audit logging across the platform. It captures all inbound API calls for audit, compliance, and operational troubleshooting purposes.

### 4.8 Core Data Stores

[Content unchanged from v2.1, with extensions.]

⚠️ **Storage pattern note (extended 2026-04-30):** Multiple components now confirmed with non-standard datasource patterns within the Aurora model:
- **COMP-37 DocumentManagementService** — only WDP component using AWS S3 and DynamoDB as primary data stores. Also has two PostgreSQL datasources (NAP and WDP) for desk-blanking column-level updates. The S3 + DynamoDB + dual-PostgreSQL pattern is unique to COMP-37.
- **COMP-22 DisputeService *(added 2026-04-28)*** — HikariCP pools unconfigured on both datasources; Spring Boot defaults apply (`maximumPoolSize=10`).
- **COMP-26 QuestionnaireService *(added 2026-04-28)*** — uses `DriverManagerDataSource` (no HikariCP, no application-tier pool). Every JPA call opens a new JDBC connection. Architect decision pending — ADR-CAND-042.
- **COMP-32 RulesService *(added 2026-04-28)*** — uses `DataSourceBuilder.create().build()` with no explicit pool tuning on either of two PostgreSQL datasources.
- **COMP-30 UserQueueSkillService *(added 2026-04-28)*** — dual-datasource design with `ukTransactionManager` (UK / nap) and `usTransactionManager` (US / wdp, `@Primary`). DEC-021 second offender — see §4.5.
- **COMP-02 UAMS *(added 2026-04-29)*** — also writes to AWS S3 directly (separate from COMP-37 path) for bulk onboarding. Hardcoded `eu-west-2` region. See §11.1 / WDP-INTEGRATIONS.md §7.2.

These patterns should not be replicated without explicit architectural review.

⚠️ **Outbox writers extended (2026-04-30):** `wdp.outgoing_event_outbox` is now confirmed as a **5-channel shared outbox** (was 3 in v2.1):
- EXPIRY_EVENTS (COMP-17, `WCSEEXPC`)
- GP_EVENTS (COMP-41, `WNEC`)
- BEN_EVENTS (COMP-42, `WBENC`) — *added 2026-04-29*
- CORE_EVENTS (COMP-43, `PCSECRTC`)
- EXPIRY_BATCH (COMP-51, `WCSEEXPB`) — *added 2026-04-25; ⚠️ terminal-write-only with no consumer identified — see §10.2*

⚠️ **NEW Case Expiry Subsystem coordination surface (2026-04-30):** `wdp.case_expiry` is shared between COMP-17 (writer half — Kafka consumer maintaining the table) and COMP-51 (reader half — scheduled batch acting on past-due rows). Coordination is **operational-only — no row-level lock, no version column, no SELECT FOR UPDATE**. See WDP-DECISIONS.md ADR-CAND-027.

---

## 5. Inbound Processing

[Existing v2.1 narrative and Mermaid retained.]

### 5.1 NAP/WPG Path (Current) ⚠️ Refined 2026-04-29

[Existing v2.1 content retained.]

⚠️ **(2026-04-29) COMP-04 enrichment chain — alternative, not sequential.** v1.0/v2.0 narrative implied a sequential three-step enrichment (CaseManagement → FraudSwitch → DisplayCodeService). Source verification confirms the chain is **alternative**:
- CaseManagement is tried first
- FraudSwitch only runs when CaseManagement returns null
- DisplayCodeService only runs on the GUARPAY1 / GUARPAY4 + fraudIndemnified branch

Bypass condition (`enrichment_failure=true OR function_code=603`) is honoured **only on POST `/event`**. Case-update and outcome paths do not pass through `EventBusinessValidator.validateRequest`. The `enrichmentFailure` field on the outbound `NapEvent` is a **pass-through copy** of the inbound flag — COMP-04 never sets it itself.

⚠️ **(2026-04-29) NAP path now has TWO REST surfaces for manual operator reprocessing:**
- **COMP-05 NAPDisputeEventProcessor** — JWT-authenticated `POST /event` for inbound NAP error reprocessing (Ops Portal access path).
- **COMP-39 NAPOutcomeProcessor** — JWT-authenticated `POST /event` for outbound NAP error reprocessing.

Both endpoints drive the same business pipeline as the Kafka path with **no per-record locking**. A manual reprocess POST and a Kafka prior-error scan can race on the same record. See ADR-CAND-037.

🔴 **NEW MATERIAL DEFICIENCY (2026-04-30) — CVV-at-rest in NAP path.** PCI-DSS 3.2.1 deficiency:
- **COMP-04** logs CVV via Lombok-generated `NapEvent.toString()` at INFO before Kafka publish.
- **COMP-05** persists CVV at rest in `NAP.DISPUTE_EVENT_CONSUMER_ERROR.C_CVV` and inside `C_KAFKA_EVENT` raw JSON.

Together these constitute a CVV-on-disk path. Architect decision required (remediate or document approved exception). See WDP-NFRS.md RISK-084 / WDP-DECISIONS.md ADR-CAND-023 / WDP-INTEGRATIONS.md §2.4.

🔴 **(2026-04-29) Cross-component shared error-table consumption hazard.** `NAP.DISPUTE_EVENT_CONSUMER_ERROR` has **four writers** confirmed: COMP-05 (primary), COMP-23 (NAP create path blind-merge), COMP-24 (NAP conditional outbox), COMP-39 (outbound NAP write). Discriminator: `C_ACQ_PLATFORM` + `C_EVENT_TYPE`. Both COMP-05 and COMP-39 prior-error scans query without `C_EVENT_TYPE` filter — rows written by either consumer (plus COMP-23 / COMP-24) may be reprocessed through the wrong outbound pipeline. See RISK-085 / ADR-CAND-024.

### 5.2 Card Network Batch Path

[Content unchanged from v2.1.]

### 5.3 File-Based Inbound Path

[Content unchanged from v2.1.]

---

## 6. Kafka Event Bus

[Sections 6.1, 6.2, 6.3 content unchanged from v2.1, with extensions below.]

⚠️ **(2026-04-30) `wdp.outgoing_event_outbox` now 5-channel shared outbox** — see §4.8 and WDP-DECISIONS.md DEC-002 refinement. New channels: BEN_EVENTS (COMP-42), EXPIRY_BATCH (COMP-51, terminal-write-only).

⚠️ **(2026-04-29) COMP-04 partition-key per-endpoint variation:** `merchantId` on case-update and outcome paths; `cardAcceptorCodeId` on POST `/event` (new-dispute path) only. See WDP-DECISIONS.md DEC-003 deviation map.

⚠️ **(2026-04-30) CommonErrorHandler list extended to 10 components.** Empty anonymous `CommonErrorHandler{}` registered platform-wide on multiple consumers. Combined with `ErrorHandlingDeserializer` and pre-ACK, deserialisation exceptions are silently swallowed. Now confirmed on: COMP-05, 14, 15, 16, 17, 18, 39, 40, 41, 42, 43. See WDP-NFRS.md RISK-025 and WDP-DECISIONS.md ADR-CAND-003.

⚠️ **(2026-04-28) COMP-22 DisputeService is NOT a Kafka producer at runtime.** Wired but commented out (commit `c29018cd`, Shringi Nitin (WP), 2025-08-08, message "code changes (#93)"). No in-source rationale.

---

## 7. Event Consumers

### 7.1 Inbound Consumers

[Content unchanged from v2.1.]

### 7.2 Processing Consumers

[Content unchanged from v2.1.]

⚠️ **NEW (2026-04-30) — Case Expiry Subsystem.** This is a logical sub-grouping rather than a new layer in the topology:

**Writer half — COMP-17 CaseExpiryUpdateConsumer.** Kafka consumer on `case-action-events`; consumes deadline-update events from COMP-18 NotificationOrchestrator (Filter 1 EXPIRY_EVENT routing) and maintains `wdp.case_expiry` (upsert on event arrival; closure-delete on cancellation).

**Reader half — COMP-51 CaseExpiryProcessor (NEW).** Standalone Spring Batch Deployment in `gcp-case-expiry-processor-batch`. Trigger is a Spring `@Scheduled` cron (NOT a Kubernetes CronJob). Scans `wdp.case_expiry` for past-due rows; calls 7 distinct upstream services (IDP, Case Action, Case Management, Expiry Rules, Update Action, Accept, Add Action). Hand-rolled retry mechanism via `wdp.case_expiry.i_retry_count` (max 3); after exhaustion, writes a row to `wdp.outgoing_event_outbox` with `channel_type=EXPIRY_BATCH`, `created_by=WCSEEXPB`, `status=ERROR` direct.

⚠️ **(2026-04-30) Two architectural risks in the subsystem:**
- **Shared-write race on `wdp.case_expiry`** — COMP-17 and COMP-51 share the table with no row-level lock, no version column, no SELECT FOR UPDATE. See ADR-CAND-027.
- **EXPIRY_BATCH outbox is terminal-write-only** — COMP-12 Scheduler3 reads only FAILED and PENDING_DEFERRED, so COMP-51's `status=ERROR` rows have **no platform consumer**. See ADR-CAND-026.

### 7.3 Outbound Integration Consumers ⚠️ Refined 2026-04-29

[Existing v2.1 NotificationOrchestrator content preserved.]

#### NAP Outcome Processor

[Existing v2.1 content preserved.]

⚠️ **(2026-04-29) Manual reprocess REST endpoint surface confirmed.** COMP-39 hosts JWT-authenticated `POST /event` for manual NAP outbound error reprocessing (Ops Portal access path). This resolves the long-standing question of which component hosts the manual reprocessing surface — **it is COMP-39, not COMP-05**. See §5.1 for the COMP-05 inbound reprocess sibling.

⚠️ **(2026-04-29) NAP-DPS authentication gap (RISK-179).** COMP-39 sets no Authorization header on outbound NAP-DPS calls; no client certificate is loaded. `napcacrt.jks` is an unreferenced orphan in the COMP-39 repo. Authentication is handled outside the component at Ingress / service mesh / network layer — pending team confirmation.

⚠️ **(2026-04-29) COMP-39 confirmed sole `@KafkaListener` on `internal-integration-events`.** Therefore COMP-39 is **NOT** the consumer of COMP-24's `${kafka.topic}` ActionEvent topic; that consumer remains unidentified.

#### VisaResponseQuestionnaire

[Content unchanged from v2.1.]

### 7.4 Notification Consumers

[Sections for ThirdPartyNotificationConsumer, BEN Consumer, EDIA Consumer (Planned), CoreNotificationConsumer all unchanged from v2.1.]

⚠️ **(2026-04-29) BEN Consumer (COMP-42) refinements:**
- Confirmed **fourth writer of `wdp.outgoing_event_outbox`** (`channel_type=BEN_EVENTS`, `created_by=WBENC`).
- BEN Product REST API is a companion contract alongside the BEN MSK Kafka publish — `Bearer ${ben_product_license}` static license-key auth, Spring Retry 3 × 1000ms, no timeout. See WDP-INTEGRATIONS.md §4.3.
- Outbound BEN partition key is `merchantId` from `CaseSearchResponse` — DEC-003 compliant.
- PUBLISHED-orphan crash window Step 4 → Step 13 — same class as RISK-040 (COMP-41 Signifyd). See RISK-192.

---

## 8. Outbound Processing

### 8.1 Card Network Direct API Calls

[Content unchanged from v2.1 — already includes the AMEX/DISCOVER + MC CHI silent no-op note and the NAP-publish split-brain consequence.]

### 8.2 Visa Questionnaire Retrieval Flow

[Content unchanged from v2.1.]

### 8.3 NAP Outcome Flow

[Content unchanged from v2.1.]

### 8.4 Notification Targets

[Content unchanged from v2.1.]

### 8.5 File-Based Outbound

[Content unchanged from v2.1.]

⚠️ **(2026-04-28) COMP-22 SFG SFTP NAP fallback path.** Distinct from the file-generation outbound path described in §8.5 — used by COMP-22 DisputeService for NAP-platform document delivery when the primary REST upload to DocumentManagementService fails. Fully `@Async` end-to-end; HTTP 200 returned to caller before SFTP write completes. Filename collision risk. See WDP-INTEGRATIONS.md §6.5 / WDP-NFRS.md RISK-133, RISK-134.

---

## 9. Acquiring Platform Integration

[Content unchanged from v2.1.]

---

## 10. Cross-Cutting Concerns

### 10.1 PAN Encryption & Security Boundary ⚠️ Extended 2026-04-30

[Existing v2.1 content preserved.]

⚠️ **(2026-04-29) DEK rotation interval correction:** Prior platform documentation stated a "6-hour DEK cache." Source verification confirms this is **incorrect**. Actual rotation interval is **days** via `${dek_rotation_interval_days}`. See §4.7 / §11.1 / WDP-DECISIONS.md DEC-008.

⚠️ **(2026-04-29) Decrypt `@Transactional` brackets KMS network call.** COMP-35 holds a Hikari connection during the KMS round-trip on every decrypt request. Pool-exhaustion risk under sustained decrypt load + KMS slowdown. See RISK-172 / ADR-CAND-035.

⚠️ **(2026-04-29) Multi-pod DEK rotation race.** No distributed lock; concurrent COMP-35 pods may attempt rotation simultaneously. Source code carries an explicit comment acknowledging this. See RISK-173 / ADR-CAND-036.

🔴 **NEW MATERIAL DEFICIENCY (2026-04-30) — CVV-at-rest in COMP-04 + COMP-05.** PCI-DSS 3.2.1 Requirement 3.2 prohibits CVV storage after authorisation **regardless of encryption status**. Two confirmed paths:
- **COMP-04** logs CVV via Lombok `NapEvent.toString()` at INFO before Kafka publish.
- **COMP-05** persists CVV at rest in `NAP.DISPUTE_EVENT_CONSUMER_ERROR` (`C_CVV` column AND `C_KAFKA_EVENT` JSON).

Together these constitute a CVV-on-disk path. **Encryption does not make CVV-at-rest compliant.** Architect decision required. See RISK-084 / ADR-CAND-023 / DEC-019 (third confirmed exception class — alongside DEC-019 (1) PostgreSQL clear PAN COMP-23 and DEC-019 (2) DB2 clear PAN COMP-43).

⚠️ **(2026-04-30) COMP-04 base64 file content in logs.** `UploadDocumentRequest.toString()` (Lombok) surfaces full base64 file content at controller entry. Same family as the CVV finding but distinct subject. See RISK-116 / ADR-CAND-032.

⚠️ **(2026-04-29) COMP-04 unauthenticated SecurityConfig — `/**` whitelist.** All endpoints unauthenticated at app level; auth relies entirely on Ingress / network controls. See §3.4.

### 10.2 Resilience Patterns ⚠️ Extended 2026-04-30

[Existing v2.1 content preserved.]

⚠️ **(2026-04-30) Strengthened evidence base.** DEC-014 VOID is now confirmed across all 38 source-verified components (was 20 in v2.1). New additions: COMP-01 (blocking RestTemplate on Netty event-loop), COMP-02 (bare RestTemplate for IdP and S3), COMP-03 (no JDBC query timeout on either datasource), COMP-04 (3 RestTemplate instances all default-constructor; one shadowed locally), COMP-05, COMP-06, COMP-22 (HikariCP and Tomcat at defaults), COMP-25 (no RestTemplate bean — `new RestTemplate()` per call), COMP-26, COMP-30, COMP-32, COMP-35 (KMS round-trip in `@Transactional`), COMP-39, COMP-42, COMP-51. See WDP-DECISIONS.md DEC-014 evidence list.

⚠️ **(2026-04-30) Empty `CommonErrorHandler{}` list extended to 10 components.** Now confirmed: COMP-05, 14, 15, 16, 17, 18, 39, 40, 41, 42, 43. Distinct silent-loss class from pre-ACK offset window. See RISK-025 / ADR-CAND-003.

⚠️ **(2026-04-30) NEW — `minReadySeconds` platform-wide misplacement (RISK-083 / ADR-CAND-030).** Eight components confirmed to place `minReadySeconds: 30` under `spec.template.spec` instead of `spec` — silently ignored by Kubernetes. Affected: COMP-03, 05, 08, 09, 12, 25, 26, 28, 34, 40 (10 components). The intended rolling-update stability gate is not actually applied. Pattern is a copy-paste-class defect — likely replicated across other WDP manifests not yet audited. DevOps remediation pass + manifest-lint rule recommended.

⚠️ **(2026-04-30) NEW — Hand-rolled retry counter persistence pattern.** Multiple components use hand-rolled retry counters persisted to source tables instead of Spring Retry framework. COMP-51 increments `wdp.case_expiry.i_retry_count` (max 3); COMP-07/08/09 use similar patterns on `chbk_outbox_row.retry_count`. Spring Retry on classpath but unused in COMP-51. See ADR-CAND-055.

⚠️ **(2026-04-30) NEW — EXPIRY_BATCH outbox terminal-write-only (RISK-090).** COMP-51 writes `status=ERROR` rows to `wdp.outgoing_event_outbox` with `channel_type=EXPIRY_BATCH`. COMP-12 Scheduler3 reads only FAILED and PENDING_DEFERRED — these rows have no platform consumer. Architect decision required (define consumer or accept as audit-only). See ADR-CAND-026.

### 10.3 Observability ⚠️ Extended 2026-04-30

[Existing v2.1 content preserved.]

⚠️ **(2026-04-30) `v-correlation-id` propagation gaps confirmed across multiple components:**
- **COMP-01** — always null; `RequestCorrelation.setId()` never called; `ThreadLocal` incompatible with reactive WebFlux threading model.
- **COMP-17** — not propagated on IDP token call (only on case-search call).
- **COMP-22** — not propagated on any outbound REST call. Interceptor places it in MDC for local logs only.
- **COMP-51** — generates a fresh random UUID per outbound REST call (anti-pattern). End-to-end audit trail across the 4–5 calls per record cannot be reconstructed from headers alone.

End-to-end distributed tracing is broken at multiple service boundaries. Platform-wide remediation candidate — see RISK-089 / ADR-CAND-044, ADR-CAND-054.

⚠️ **(2026-04-30) Kafka-path MDC enrichment gap.** Multiple Kafka consumers have no MDC enrichment (COMP-17, 18, 40, 43, 51). HTTP path uses `HttpInterceptor`; Kafka path has no equivalent. Per-message log correlation depends entirely on OTel agent context. See ADR-CAND-057.

⚠️ **(2026-04-29) `/actuator/prometheus` JWT-protection widespread.** Confirmed on COMP-21, COMP-25, COMP-29, COMP-35. Scrape-side configuration must carry JWT or platform-wide whitelisting needed. See RISK-140 (COMP-25-specific).

### 10.4 PCI-DSS & Compliance ⚠️ Refreshed 2026-04-30

| Framework | Scope | Status |
|---|---|---|
| PCI-DSS 3.2.1 | Full platform — all cardholder data flows | ✅ Active — ⚠️ **(2026-04-30) NEW MATERIAL DEFICIENCY:** see RISK-084 (CVV at rest in COMP-05 + CVV in logs in COMP-04). Architect decision pending — see DEC-019 (third confirmed exception) and ADR-CAND-023. |
| SOC 2 Type II | Security, availability, processing integrity | ✅ Active |
| GDPR | Data subject rights, data privacy, right to erasure | ✅ Active |
| CCPA | California resident data rights | ✅ Active |
| SOX | Immutable audit trail for financial communications | ✅ Active |

[Audit & Retention table unchanged from v2.1.]

---

## 11. Deployment Context

### 11.1 AWS Infrastructure ⚠️ Refined 2026-04-29

[Existing v2.1 content preserved, with corrections below.]

**AWS KMS (Key Management Service)** — Manages the Customer Master Key (CMK) used to encrypt Data Encryption Keys (DEKs) that protect PAN data. Uses FIPS 140-2 Level 3 validated hardware security modules. ⚠️ **(2026-04-29)** DEK rotation interval is **days** via `${dek_rotation_interval_days}` (correction from prior 6-hour figure). EncryptionService (COMP-35) is the sole KMS caller — see WDP-INTEGRATIONS.md §6.2.

**AWS ElastiCache** — ⚠️ **(2026-04-29) Rewritten:** ElastiCache Redis hosts the `wdpinternalidptoken:token` JWT token cache. **TokenService (COMP-36) is read-only on Redis** — performs HGET only; zero write operations exist anywhere in COMP-36 source. The Redis hash is populated by an UNKNOWN EXTERNAL COMPONENT not present in any audited WDP repo. External writer identity is an open question (RISK-class informational). Two-layer cache pattern: ElastiCache Redis (primary) + in-memory OAuth2 store (secondary).

**AWS Secrets Manager** — Stores the HMAC key used for HPAN generation. ⚠️ **(2026-04-29)** Now formalised as a distinct enterprise integration entry — see WDP-INTEGRATIONS.md §6.9. Sole caller COMP-35, startup-only.

⚠️ **(2026-04-29) AWS S3 region inconsistency.** Three different region values now confirmed across S3-using components:
- COMP-37 (DocumentManagementService) — region unspecified in source (assumes `us-east-1`)
- COMP-11 (FileProcessor) — `S3ClientConfiguration` hardcodes `us-east-2`
- COMP-02 (UAMS) — bulk onboarding `${wdp_entity_file}` bucket hardcodes `eu-west-2`

Platform-primary region per §11 / WDP-NFRS Section 5.6 is `us-east-1`. Architect decision pending on whether the `eu-west-2` and `us-east-2` instances are intentional regional choices or copy-paste drift.

⚠️ Detailed infrastructure configuration — instance sizes, broker counts, replica counts, storage provisioning, retention settings — to be documented in dedicated infrastructure documentation.

### 11.2 On-Premise Components — WDP Owned

[Content unchanged from v2.1.]

### 11.3 On-Premise Components — Enterprise Shared Services

[Content unchanged from v2.1.]

### 11.4 Infrastructure Overview

[Content unchanged from v2.1.]

---

## 12. Component Status Registry

This section provides a single reference table covering all WDP components with their current production status. Use this as the definitive source for understanding what is live, what is planned, and what is in progress.

⚠️ **(2026-04-30) Source-verification status:** **38 of 51 components** have undergone source-verified correction passes between 2026-04-18 and 2026-04-30. Detailed component-by-component status is maintained in **WDP-COMP-INDEX.md v2.2** (51 components total; 4 audit-pending — COMP-29, 31, 34, 38). The tables below show production-deployment status only.

**Status legend:**
- ✅ Production — component is live and in production
- 🔄 In Progress — component is being built or partially in production
- 🔴 Planned — component is planned but not yet started
- ⚠️ Migration Planned — component is live but has planned migration work

---

### 12.1 UI & Access Layer

[Content unchanged from v2.1, with row-level updates.]

| Component | Status | Notes |
|---|---|---|
| WDP Merchant Portal | ✅ Production | |
| WDP Ops Portal | ✅ Production | |
| Disputes Section | ✅ Production | Both portals |
| Queues Section | ✅ Production | Ops Portal only |
| User Management Section | ✅ Production | Both portals |
| Org Management Section | ✅ Production | Both portals |
| Dashboard Section | 🔴 Planned | Dispute analytics for merchants — not yet developed |
| Akamai | ✅ Production | CDN & edge security — Merchant Portal and B2B path |
| APIGEE | ✅ Production | B2B / system-to-system path only |
| API Gateway | ✅ Production | Single entry point — ⚠️ **CORE/VAP/LATAM gateway-level authorization absent (RISK-class)**; ⚠️ blocking RestTemplate on Netty event-loop (RISK-093) |
| IDP | ✅ Production | Shared enterprise OAuth 2.0 |
| UserAccessManagementService | ✅ Production | ⚠️ **DEC-021 wrong-TM scope expanded to 7 methods (RISK-010 🔴)**; 4-path summary in §3.4 / §4.5 |

---

### 12.2 WDP Core Services

[Content unchanged from v2.1, except as noted below.]

| Component | Status | Notes |
|---|---|---|
| AcceptService | ✅ Production | ⚠️ NAP split-brain on MC CHI / AMEX / DISCOVER (RISK-028 / ADR-CAND-001) |
| ContestService | ✅ Production | |
| ChargebackService | ✅ Production | Externally exposed via APIGEE; 38 unprotected outbound call sites |
| **DisputeService** | ✅ Production | ⚠️ **(2026-04-28) Confirmed source-verified zero writes** — Kafka producer wired but commented out (commit `c29018cd`) |
| CaseManagementService | ✅ Production | |
| CaseActionService | ✅ Production | |
| NotesService | ✅ Production | ⚠️ **(2026-04-28) Mid-batch Kafka orphan deterministic split-brain (RISK-139)** |
| **QuestionnaireService** | ✅ Production | ⚠️ **(2026-04-28) No `@Transactional` posture; `DriverManagerDataSource`; ADR-CAND-042 / 043** |
| DocumentManagementService | ✅ Production | 6th publisher of `business-rules` |
| CaseSearchService | ✅ Production | |
| **DisplayCodeService** | ✅ Production | ⚠️ Permission-shape divergence between `/search` (11 flags) and `/privileges` (17 flags) — RISK-149 |
| FaxQueueService | ✅ Production | Ops Portal only — fax detail section pending |
| **UserQueueSkillService** | ✅ Production | ⚠️ **(2026-04-28) DEC-021 second offender (RISK-class HIGH)** |
| BusinessRulesService | ✅ Production | Rule management only — does not execute rules |
| **RulesService** | ✅ Production | ⚠️ **(2026-04-28) Pure read-only confirmed; `migrationStatus="N"` undocumented prod kill-switch (RISK-164 / ADR-CAND-049)** |
| OrgManagementService | ✅ Production | GitHub repo not found |
| CoreHierarchyAuthorizationService | ✅ Production | ⚠️ **9 endpoints** (was 6); PIN+CORE scope (was PIN-only); `validateOrgId` method body absent |
| MerchantTransactionService | ✅ Production | CORE enrichment only |
| **EncryptionService** | ✅ Production | ⚠️ **DEK rotation interval days** (correction); single global dependency for all PAN ingestion (RISK-170); KMS-in-`@Transactional` (RISK-172) |
| **TokenService** | ✅ Production | ⚠️ **Read-only on Redis** (zero write ops); external writer of `wdpinternalidptoken:token` unknown |
| APILogService | ✅ Production | |

---

### 12.3 Core Data Stores

| Component | Status | Notes |
|---|---|---|
| Aurora PostgreSQL | ✅ Production | Multi-AZ — primary operational database |
| DynamoDB | ✅ Production | DocumentManagementService exclusive |
| S3 /inbound | ✅ Production | Source-specific folders per inbound source |
| S3 /staging | ✅ Production | Evidence and issuer documents |
| S3 /outbound | ✅ Production | Target-specific folders per outbound target |
| S3 Documents | ✅ Production | Evidence documents (via COMP-37) |
| **S3 UAMS bulk onboarding (`eu-west-2`)** | ✅ Production | ⚠️ **NEW (2026-04-29)** — distinct from S3 Documents path; COMP-02 direct caller; no downstream parser identified |
| **AWS ElastiCache** | ✅ Production | ⚠️ **TokenService is read-only on Redis** (correction 2026-04-29) |
| **AWS KMS** | ✅ Production | CMK for PAN encryption — ⚠️ DEK rotation interval days (correction 2026-04-29) |
| **AWS Secrets Manager** | ✅ Production | ⚠️ **NEW formalised entry (2026-04-29)** — sole caller COMP-35, startup-only |

---

### 12.4 Inbound Processing

| Component | Status | Notes |
|---|---|---|
| NAPDisputeEventService | ✅ Production | ⚠️ Migration planned to common path; ⚠️ **(2026-04-29) `/**` whitelist unauthenticated; CVV in logs (RISK-115, RISK-114)** |
| NAPDisputeEventProcessor | ✅ Production | ⚠️ Migration planned; ⚠️ **(2026-04-29) CVV at rest (RISK-084)**; manual reprocess REST surface confirmed |
| NAPDisputeDeclineBatch | ✅ Production | Decommission-scoped; ⚠️ Source-verified 2026-04-30 |
| VisaDisputeBatch | ✅ Production | Polls multiple Visa queues every 2 minutes |
| FirstChargebackBatch | ✅ Production | Polls MasterCard for first chargeback events |
| CaseFillingBatch | ✅ Production | Polls MasterCard for subsequent dispute events |
| FileProcessor | ✅ Production | Triggered by SQS — processes all inbound files |
| InboundDisputeEventScheduler | ✅ Production | Polls outbox tables — 5 schedulers, 5-channel relay |
| FileAcknowledgementProcessor | ✅ Production | Generates ACK files for Meijer, Walmart, CapitalOne |

---

### 12.5 Outbox Tables

| Component | Status | Notes |
|---|---|---|
| chbk_outbox_row | ✅ Production | Central outbox — all inbound paths converge here |
| **wdp.outgoing_event_outbox** | ✅ Production | ⚠️ **5-channel shared outbox** (was 3 in v2.1): EXPIRY_EVENTS, GP_EVENTS, BEN_EVENTS, CORE_EVENTS, **EXPIRY_BATCH (terminal-write-only — no consumer)** |
| wdp.bre_orchestration_outbox | ✅ Production | Shared by COMP-12 Scheduler4 and COMP-18 |
| file_job | ✅ Production | File-level processing status tracker |
| file_evidence | ✅ Production | Evidence document metadata + S3 staging path |
| **wdp.file_generation_event** | ✅ Production | ⚠️ Corrected name (was incorrectly `file_notifications` in earlier docs) — sole writer COMP-18 |
| **wdp.case_expiry** | ✅ Production | ⚠️ **NEW (2026-04-30) — Case Expiry Subsystem coordination surface (COMP-17 writer half + COMP-51 reader half)**; no row-level lock |

---

### 12.6 Kafka Event Bus

| Component | Status | Notes |
|---|---|---|
| AWS MSK | ✅ Production | 3 brokers across 3 AZs |
| nap-dispute-events | ✅ Production | |
| new-case-events | ✅ Production | |
| case-evidence-events | ✅ Production | |
| **business-rules** | ✅ Production | ⚠️ **6 confirmed publishers (resolved 2026-04-25)**: COMP-12 Scheduler4, COMP-15, COMP-23, COMP-24, COMP-25, COMP-37 |
| outgoing-events | ✅ Production | |
| internal-integration-events | ✅ Production | |
| case-action-events (expiry) | ✅ Production | Cert env uses distinct topic |
| external-request-events | ✅ Production | |
| core-request-events (DB2) | ✅ Production | |
| **BEN-owned MSK cluster topic** | ✅ Production | ⚠️ Separate cluster — not WDP MSK; COMP-42 publishes; distinct SASL credentials |
| EDIA events | 🔴 Planned | Enterprise Kafka topic on EDIA platform |

---

### 12.7 Event Consumers

| Component | Status | Notes |
|---|---|---|
| CaseCreationConsumer | ✅ Production | Does not handle NAP disputes currently |
| EvidenceConsumer | ✅ Production | |
| BusinessRulesProcessor | ✅ Production | Makes direct DB calls — does not call BusinessRulesService |
| **CaseExpiryUpdateConsumer (COMP-17)** | ✅ Production | ⚠️ **Writer half of Case Expiry Subsystem (NEW grouping 2026-04-30)** |
| NotificationOrchestrator | ✅ Production | Routes by business logic in code |
| **NAP Outcome Processor (COMP-39)** | ✅ Production | ⚠️ Migration planned to EDIA route; ⚠️ **(2026-04-29) Manual reprocess REST endpoint confirmed; NAP-DPS auth gap (RISK-179)** |
| VisaResponseQuestionnaire | ✅ Production | ⚠️ allocarb iteration partial-failure (RISK-185) |
| ThirdPartyNotificationConsumer | ✅ Production | Delivers to SignifyD via REST API; ⚠️ JustAI planned only for outbound |
| **BEN Consumer** | ✅ Production | ⚠️ **Kafka publish to BEN-owned MSK cluster** (corrected from "via webhook"); 4th writer of `wdp.outgoing_event_outbox` |
| EDIA Consumer | 🔴 Planned | WDP owned — converts to EDIA enterprise format |
| CoreNotificationConsumer | ✅ Production | Delivers to CORE via DB2; clear PAN exception (RISK-035) |
| **CaseExpiryProcessor (COMP-51)** | ✅ Production | ⚠️ **NEW (2026-04-30) — Reader half of Case Expiry Subsystem; not a Kafka consumer (Spring Batch)** |

---

### 12.8 Outbound File Generation

[Content unchanged from v2.1.]

---

### 12.9 Notification Targets

| Target | Status | Integration |
|---|---|---|
| SignifyD | ✅ Production | REST API via ThirdPartyNotificationConsumer; also a partner identity in COMP-21 inbound |
| **JustAI (inbound partner identity in COMP-21)** | ✅ Production | REST HTTPS — active in COMP-21 source as `JUSTTAI` consumer name |
| **JustAI (outbound notification target in COMP-41)** | 🔴 Planned | Not in COMP-41 codebase; Spring Retry imports dead |
| **BEN** | ✅ Production | ⚠️ **Kafka publish to BEN-owned MSK cluster** (corrected from "Webhook"); separate SASL/JAAS credentials |
| CORE | ✅ Production | DB2 via CoreNotificationConsumer |
| NAP | 🔴 Planned (via EDIA) | Currently direct API via COMP-39 |
| LATAM | 🔴 Planned | Via EDIA platform |
| VAP | 🔴 Planned | Via EDIA platform |

---

### 12.10 Acquiring Platform Integration

[Content unchanged from v2.1.]

---

### 12.11 On-Premise Infrastructure

[Content unchanged from v2.1.]

---

### 12.12 Planned Work Summary ⚠️ Refreshed 2026-04-30

| # | Item | Section | Type |
|---|---|---|---|
| 1 | Dashboard Section (UI) | 3.1 | New feature |
| 2 | NAP inbound migration to chbk_outbox_row → CaseCreationConsumer | 5.1 / 9.3 | Migration |
| 3 | NAP outbound migration from direct API to EDIA route | 8.3 / 9.3 | Migration |
| 4 | EDIA Consumer | 7.4 | New component |
| 5 | EDIA events Kafka topic | 6.2 | New topic |
| 6 | NAP acquiring platform via EDIA | 9.3 | New integration |
| 7 | LATAM acquiring platform integration | 9.4 | New integration |
| 8 | VAP acquiring platform integration | 9.5 | New integration |
| 9 | NYCE File Generation Processor | 8.5 | New component |
| 10 | AMEX/Discover SFTP outbound | 8.5 | New integration |
| 11 | Circuit breaker strategy evaluation | 10.2 | Architectural decision |
| 12 | CORE DB2 → EDIA migration | 9.2 | Future consideration |
| **13** | **CVV-at-rest remediation (COMP-04 + COMP-05)** | **10.1 / 10.4** | **🔴 PCI-DSS material deficiency — architect decision pending (ADR-CAND-023)** |
| **14** | **EXPIRY_BATCH outbox consumer definition (COMP-51)** | **4.8 / 7.2** | **Architectural decision (ADR-CAND-026)** |
| **15** | **Case Expiry Subsystem coordination (COMP-17 + COMP-51 shared write)** | **4.8 / 7.2** | **Architectural decision (ADR-CAND-027)** |
| **16** | **DEC-021 scope expansion remediation (COMP-02 7 methods + COMP-30 second offender)** | **3.4 / 4.5** | **🔴 Defect remediation (ADR-CAND-025, ADR-CAND-033)** |
| **17** | **CORE/VAP/LATAM gateway-level authorization gap remediation** | **3.3 / 3.4** | **🔴 Architectural decision (ADR-CAND-029)** |
| **18** | **COMP-04 unauthenticated SecurityConfig — Spring Security JWT or document infrastructure-layer-only auth** | **3.4 / 5.1** | **🔴 Architectural decision (ADR-CAND-031)** |
| **19** | **COMP-01 blocking RestTemplate on Netty — migrate to WebClient or accept** | **3.3 / 10.2** | **🔴 Architectural decision (ADR-CAND-028)** |
| **20** | **`minReadySeconds` platform-wide remediation pass + manifest-lint rule** | **10.2 / 11** | **DevOps remediation (ADR-CAND-030)** |

---

## Open Discussion Points & Follow-Ups ⚠️ Refreshed 2026-04-30

The following items have been flagged during the architecture review and require further discussion or confirmation. Status updates from the 2026-04-28/29/30 source-verification reconciliation pass are noted.

| # | Topic | Section | Priority | Status |
|---|---|---|---|---|
| 1 | Discover vs DiscoverHybrid — detailed file flow differences | 5.4 | High | Open |
| 2 | Amex vs AmexHybrid — detailed file flow differences | 5.4 | High | Open |
| 3 | File content classification per source | 5.4 | High | Open |
| 4 | Acknowledgement file rules — which sources require ACK and which do not | 5.4 | High | Open |
| 5 | S3 folder key structure and naming conventions per source and target | 5.4 | Medium | Open |
| 6 | File-only network issuer documents — confirm if sent in separate files via DM Mainframe | 5.3 | Medium | Open |
| 7 | Visa & MasterCard issuer document retrieval — confirm at which processing stage and which component | 5.3 / 7 | High | Open |
| 8 | Circuit breaker strategy — evaluate and document as future architectural decision | 10.2 | Medium | Open — see ADR-CAND-007 (No K8s probes — same hardening sprint) |
| 9 | business-rules topic publisher — confirm which component publishes | 6.2 | — | ✅ Resolved 2026-04-25 — six confirmed publishers |
| 10 | case-action-events partition key — confirm merchant_id or case_id | 6.2 | Low | ✅ Resolved 2026-04-25 — pass-through `RECEIVED_KEY` |
| 11 | Consumer group names — confirm exact names at component level | 6.3 | Low | ✅ Resolved across 9 reconciled consumers |
| 12 | UserAccessManagementService — detailed behaviour to be confirmed | 3.4 | Medium | ✅ **Resolved 2026-04-29 — 4-path summary documented in §3.4 / §4.5** |
| 13 | Fax functionality — dedicated section needed under Queues | 4.3 | Medium | Open |
| 14 | CORE DB2 → EDIA migration — capture as open architectural decision | 9.2 | Low | Open |
| 15 | Observability tooling — document in dedicated operational architecture pass | 10.3 | Medium | Partial — WDP-OBSERVABILITY-ARCHITECTURE.md exists; not yet integrated into Section 10.3 |
| 16 | Compliance frameworks — verify and update current list | 10.4 | Medium | ✅ Captured in WDP-NFRS.md v2.2 Section 3 |
| 17 | Per-consumer outbox tables — document at component level | 7 | High | ✅ Captured in WDP-DB.md v2.2 Section 4 |
| 18 | NAP CB911 migration timeline and completion criteria | 9.3 | Medium | Open |
| 19 | AcceptService NAP split-brain (MC CHI / AMEX / DISCOVER) | 8.1 | High | Open — ADR-CAND-001 |
| 20 | COMP-43 DB2 clear PAN — extend DEC-019 or remediate? | 4.8 | High | Open — ADR-CAND-004 |
| 21 | COMP-12 production replica count > 1 produces guaranteed duplicate Kafka publishes | 11 | High | Open — RISK-038 |
| **22** | **CVV-at-rest in COMP-04 logs + COMP-05 error tables — PCI-DSS 3.2.1 material deficiency** | **5.1 / 10.1 / 10.4** | **🔴 HIGH** | **Open — ADR-CAND-023 (NEW 2026-04-30); architect decision required** |
| **23** | **Cross-component shared error-table consumption hazard on `NAP.DISPUTE_EVENT_CONSUMER_ERROR` — uniform decision required for COMP-05 + COMP-39 prior-error scans** | **5.1 / 7.3** | **🔴 HIGH** | **Open — ADR-CAND-024 (NEW 2026-04-30)** |
| **24** | **EXPIRY_BATCH outbox channel terminal-write-only — define consumer or accept as audit-only sink** | **4.8 / 7.2** | **🔴 HIGH** | **Open — ADR-CAND-026 (NEW 2026-04-30)** |
| **25** | **Case Expiry Subsystem coordination — shared write to `wdp.case_expiry` between COMP-17 (writer) and COMP-51 (reader) with no row-level lock** | **4.8 / 7.2** | **MEDIUM** | **Open — ADR-CAND-027 (NEW 2026-04-30)** |
| **26** | **DEC-021 scope expansion remediation — 7 wrong-TM methods in COMP-02 + COMP-30 second offender. Multi-datasource service-level `@Transactional` binding contract.** | **3.4 / 4.5** | **🔴 HIGH** | **Open — ADR-CAND-025, ADR-CAND-033 (NEW 2026-04-30)** |
| **27** | **CORE/VAP/LATAM gateway-level authorization gap — these platform types receive no authorization at gateway** | **3.3 / 3.4** | **🔴 HIGH** | **Open — ADR-CAND-029 (NEW 2026-04-30)** |
| **28** | **COMP-04 unauthenticated SecurityConfig (`/**` permitAll) — Spring Security JWT or document infrastructure-layer-only auth** | **3.4 / 5.1** | **🔴 HIGH** | **Open — ADR-CAND-031 (NEW 2026-04-30)** |
| **29** | **COMP-01 blocking `RestTemplate` on Netty event-loop with no timeouts — migrate to `WebClient` or accept latency floor** | **3.3 / 10.2** | **🔴 HIGH** | **Open — ADR-CAND-028 (NEW 2026-04-30)** |
| **30** | **`minReadySeconds` platform-wide misplacement (10 components confirmed) — DevOps remediation pass + manifest-lint rule** | **10.2 / 11** | **🟠 HIGH** | **Open — ADR-CAND-030 (NEW 2026-04-30)** |
| **31** | **DEK rotation interval — corrected from 6 hours to days** | **4.7 / 10.1 / 11.1** | **—** | **✅ Resolved 2026-04-29 — DEC-008 corrected; documents updated v2.2** |
| **32** | **TokenService Redis hash external writer identity** — `wdpinternalidptoken:token` is populated by an unknown component | **4.7 / 11.1** | **MEDIUM** | **Open (NEW 2026-04-29) — team confirmation needed** |
| **33** | **AWS S3 region inconsistency — three different region values across components (`us-east-1` default, `us-east-2` COMP-11, `eu-west-2` COMP-02)** | **11.1** | **MEDIUM** | **Open (NEW 2026-04-29)** |
| **34** | **`v-correlation-id` propagation gaps (COMP-01, 17, 22, 51) — distributed tracing broken at multiple service boundaries** | **10.3** | **MEDIUM** | **Open (NEW 2026-04-30) — ADR-CAND-044, ADR-CAND-054** |

---

*This document contains architecture-level content only. Implementation details, database schemas, configuration values, code patterns, and deployment specifications are maintained separately at component level.*

*v2.2 reconciled 2026-04-30 — minor topology-level clarifications: component count 50→51 (COMP-51 CaseExpiryProcessor added; grouped with COMP-17 under "Case Expiry Subsystem"); UAMS 4-path summary added; CHAS endpoint count 6→9 and scope PIN+CORE; CORE/VAP/LATAM gateway authorization gap surfaced; COMP-04 `/**` whitelist surfaced; DEK rotation interval corrected (days not 6 hours); ElastiCache rewritten (TokenService read-only); `wdp.outgoing_event_outbox` 5-channel; `wdp.case_expiry` coordination surface; CVV-at-rest PCI-DSS material deficiency (COMP-04 + COMP-05); CommonErrorHandler list extended to 10 components; minReadySeconds platform-wide misplacement (10 components); v-correlation-id propagation gaps; COMP-22 zero-writes confirmation; COMP-30 DEC-021 second offender; Component Status Registry refreshed across 8 sub-tables; Open Discussion Points: 4 resolved (12, 16 reaffirmed, 17 reaffirmed, 31 NEW-and-resolved), 13 new entries (22-34) added.*
