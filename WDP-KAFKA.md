# WDP-KAFKA.md
**Worldpay Dispute Platform — Kafka Topic Registry**
*Version: 2.1 | April 2026*
*Reconciled: 2026-04-25 — absorbed Pending Entries from WDP-CHANGE-LOG.md for COMP-07, COMP-08, COMP-09, COMP-11, COMP-12, COMP-13, COMP-14, COMP-15, COMP-16, COMP-17, COMP-18, COMP-19, COMP-20, COMP-21, COMP-23, COMP-24, COMP-27, COMP-37, COMP-41, COMP-43.*
*Component file source-verification status: 📝 DRAFT — architect confirmation pending on all 20 components.*

---

## Purpose of This Document

This is the master Kafka reference for the WDP platform. It contains:

1. **MSK Infrastructure** — broker configuration, cluster settings,
   platform-wide Kafka standards
2. **Topic Registry** — every Kafka topic in WDP with its publisher,
   consumers, partition key, payload structure, and status
3. **Consumer Group Registry** — all consumer groups, their topics,
   and offset strategies
4. **Platform Standards** — the DEC-referenced Kafka decisions that
   apply across all components

Individual component files (WDP-COMP-[NN]-*.md) contain the Kafka
contracts specific to each component. This file is the cross-cutting
view — the complete topic map for the platform.

---

## 1. MSK Infrastructure

**Platform:** AWS Managed Streaming for Apache Kafka (MSK)
**Architectural decision:** DEC-013 — Kafka (AWS MSK) as Event Streaming Platform

| Parameter | Value | Source |
|-----------|-------|--------|
| Cluster location | AWS (same region as EKS cluster) | Confirmed |
| Broker count | ⚠️ To be confirmed from infrastructure docs | PENDING |
| Availability zones | ⚠️ To be confirmed | PENDING |
| Authentication | AWS IAM (MSK IAM auth — `SASL_SSL` + `AWS_MSK_IAM` mechanism + `IAMLoginModule` + `IAMClientCallbackHandler`) | Confirmed uniformly across COMP-14, COMP-15, COMP-16, COMP-17, COMP-18, COMP-39, COMP-41, COMP-42, COMP-43 |
| BEN MSK cluster | Separate BEN-owned MSK cluster — distinct bootstrap servers, separate `${ben_sasl_config}` (BEN delivery is Kafka, NOT REST webhook) | Confirmed from COMP-42 |
| Default retention | ⚠️ To be confirmed per topic | PENDING |
| Storage scaling | One-directional — cannot reduce after scaling up | Confirmed |
| Partition strategy | Merchant-scoped (DEC-003) — deviations flagged below | Confirmed |
| Offset commit standard | Manual commit after full processing (DEC-005) — deviations flagged below | Confirmed |

⚠️ Broker count, AZ configuration, storage configuration, and per-topic
retention settings are to be confirmed from infrastructure documentation.

---

## 2. Platform-Wide Kafka Standards

These decisions apply to every Kafka interaction in WDP unless a
component explicitly deviates. All deviations are flagged in the
relevant WDP-COMP-[NN]-*.md file under Key Architectural Decisions.

| Standard | Decision | Reference |
|----------|----------|-----------|
| Partition key | merchantId on all topics — merchant-scoped isolation | DEC-003 |
| Offset commit | Manual commit after full processing completes | DEC-005 |
| Consumer failure | Database-backed error table (not Kafka DLQ topics) | Platform pattern |
| Idempotency key | UUID carried as Kafka message header `idempotency-key` (kebab-case on the wire) | Platform pattern |
| Message ordering | Guaranteed within partition (per merchantId) | DEC-003 |
| Producer idempotence | enable.idempotence=true on all Kafka producers | Per component |

---

### Confirmed DEC-005 Deviations (offset commit timing)

Every confirmed Kafka consumer in WDP uses pre-ACK or mid-flow ACK
rather than the DEC-005 standard of committing after full processing.
This is a platform-wide pattern, not an isolated exception.

| Component | Topic | Deviation detail |
|-----------|-------|-----------------|
| COMP-05 NAPDisputeEventProcessor | `nap-dispute-events` | Pre-ACK — first call in `onMessage()` |
| COMP-12 InboundDisputeEventScheduler | (producer only — outbox mark-before-send) | Mark-before-send within a single `@Transactional` — broker ACK precedes TX commit. **At-least-once with duplicate-possible crash window** (NOT at-most-once as v1.0 implied — corrected 2026-04-18). Consumer-side `idempotency-key` dedup is the intended mitigation. |
| COMP-14 CaseCreationConsumer | `new-case-events` | Pre-ACK — `KafkaConsumer.java:38` ack precedes `processKafkaNotificationEvent()` at `:43` |
| COMP-15 EvidenceConsumer | `case-evidence-events` | Pre-ACK — Step 1, before all processing. `MANUAL_IMMEDIATE` with `syncCommits=true` |
| COMP-16 BusinessRulesProcessor | `business-rules` | Pre-ACK — `KafkaConsumer.java:38` ack precedes `processRulesEvent()` at `:41` *(corrected 2026-04-18 from previously documented `:37` / `:40`)* |
| COMP-17 CaseExpiryUpdateConsumer | `case-action-events` | Inconsistent — Path A: ACK as first listener action (pre-processing); Path B: ACK mid-flow after outbox INSERT but before `case_expiry` write; header-blank path: ACK after error-row INSERT (and skipped if INSERT throws) |
| COMP-18 NotificationOrchestrator | `outgoing-events` | Mid-flow ACK — single ACK call site after Step 3d outbox INSERT, before all Step 7 publishes |
| COMP-39 NAPOutcomeProcessor | `internal-integration-events` | Pre-ACK — `acknowledgment.acknowledge()` as first call in `onMessage()` |
| COMP-40 VisaResponseQuestionnaire | `internal-integration-events` | Pre-ACK (consistent with platform pattern) |
| COMP-41 ThirdPartyNotificationConsumer | `external-request-events` | Pre-Signifyd ACK — `MANUAL_IMMEDIATE` + `syncCommits=true`. Committed after PUBLISHED outbox INSERT, before `processEvent()` and any Signifyd REST call. At-most-once delivery to Signifyd. |
| COMP-42 BENConsumer | `external-request-events` | Pre-ACK after idempotency DB check (Step 3); before CMS call (Step 6), BEN Product call (Step 7), CAS call (Step 10), BEN Kafka publish (Step 12), outbox status update (Step 13) |
| COMP-43 CoreNotificationConsumer | `core-request-events` | Pre-ACK on all paths — Path A: post outbox-ERROR write, pre-DB2; Path B: pre `processCaseActionEvent`; Path C: post outbox-PUBLISHED INSERT, pre-DB2 |

### Confirmed DEC-001 Deviations (transactional outbox)

| Component | Topic | Deviation detail |
|-----------|-------|-----------------|
| COMP-04 NAPDisputeEventService | `nap-dispute-events` | Direct publish on HTTP thread — no outbox |
| COMP-15 EvidenceConsumer | `business-rules` | Synchronous publish inside `@Transactional` via `kafkaTemplate.send(...).get()` blocking on the future — ghost-event window: broker ACK followed by a later commit-time exception emits a published-but-unpersisted event |
| COMP-16 BusinessRulesProcessor | `outgoing-events`, `internal-integration-events` | Direct synchronous publish — no outbox. Broker unavailable = permanent event loss |
| COMP-18 NotificationOrchestrator | `case-action-events`, `core-request-events`, `external-request-events` | ⚠️ PARTIAL — `wdp.bre_orchestration_outbox` is used as the outbox, but **zero `@Transactional` annotations** in the codebase. Four distinct write points (3a ERROR, 3d PUBLISHED, 6 ERROR/PENDING_DEFERRED, 7e SUCCESS/FAILED) are independent auto-commits. Outbox INSERT at Step 3d is not atomic with the Kafka offset commit at Step 4 nor with downstream publishes at Step 7. PUBLISHED orphan rows have no automatic re-drive (Scheduler4 reads only FAILED/PENDING_DEFERRED — see RISK-015). |
| COMP-19 AcceptService | `internal-integration-events` | Direct synchronous publish — no outbox. HTTP 500 on Kafka failure does not undo the case action; NAPOutcomeProcessor not notified. State permanently inconsistent on Kafka final failure. |
| COMP-20 ContestService | `internal-integration-events` | Direct synchronous publish — no outbox. CaseManagement `insertActions` commit is permanent regardless of Kafka outcome. Retry-exhaustion path writes an SNOTE but does not recover the event. |
| COMP-23 CaseManagementService | `business-rules` | Synchronous publish **inside `@Transactional` before commit** — `kafka-before-commit` pattern. Send failure rolls back the DB; broker ACK followed by a later commit-time exception emits a published-but-unpersisted event (narrow but non-zero window). *(Corrected 2026-04-23 from "publish after DB transaction commits" — source-verified.)* |
| COMP-24 CaseActionService | `business-rules`, ActionEvent topic | ⚠️ PARTIAL — `wdp.chbk_outbox_row` / `nap.DISPUTE_EVENT_CONSUMER_ERROR` inbound-outbox UPDATEs are inside the same `@Transactional` as the domain write. **BRE publish is also inside `@Transactional`** *(corrected 2026-04-23 from v1.0's "post-commit split-brain" claim)*. **ActionEvent publish is outside `@Transactional` on EP 2 / 8 / 9 — genuine post-commit split-brain when `napUpdateEvent=true`.** |
| COMP-25 NotesService | `business-rules` | Synchronous publish inside `@Transactional` boundary — not atomically coupled |
| COMP-37 DocumentManagementService | `business-rules` | Direct synchronous publish after DynamoDB write on primary upload path; no outbox. Questionnaire path (Endpoint 11) is `@Transactional(rollbackOn=Exception.class)` — partial mitigation (stronger atomicity than other publishers on this topic) but still no outbox. |
| COMP-43 CoreNotificationConsumer | (consumer side — `wdp.outgoing_event_outbox` repurposed as outbox) | Outbox INSERT on `wdpTransactionManager` commits **before** `coreDao.saveCoreCase` is invoked on `coreTransactionManager`. No XA. Crash window between outbox INSERT and DB2 write is wide open. |
| COMP-17 CaseExpiryUpdateConsumer | (consumer side — `wdp.outgoing_event_outbox` repurposed as outbox) | Dual deviation: (a) outbox repurposed as consumer-side audit/idempotency store, no Kafka publish; (b) outbox INSERT and `case_expiry` write run in **separate** transactional boundaries — outbox saves have no service `@Transactional`; `case_expiry` saves are `@Transactional REQUIRED`. Both share `wdpTransactionManager` but no method brackets both writes. |

### Confirmed DEC-003 Deviations (merchantId partition key)

| Component | Topic | Actual key | Note |
|-----------|-------|-----------|------|
| COMP-04 NAPDisputeEventService | `nap-dispute-events` | `cardAcceptorCodeId` (POST /event only); `merchantId` for other event types | Partial deviation — decommission-scoped |
| COMP-12 InboundDisputeEventScheduler | `new-case-events` | Compound: `networkCaseId + cardNetwork + platform` | Confirmed from source (both producer and consumer side) |
| COMP-12 InboundDisputeEventScheduler | `case-evidence-events` | `caseNumber` if non-blank; else `networkCaseId` | Confirmed from source |
| COMP-14 CaseCreationConsumer | `new-case-events` | (Consumer side) compound key logged as `keyNetworkCaseCardNetworkId` — not used for routing inside the consumer. Per-partition ordering is scoped to the compound key, not `merchantId`. | Consumer-side confirmed 2026-04-18 — deviation confirmed both sides |
| COMP-15 EvidenceConsumer | `business-rules` (producer) | `caseNumber` | Confirmed from source |
| COMP-16 BusinessRulesProcessor | `outgoing-events`, `internal-integration-events` | `caseNumber` | Confirmed from source |
| COMP-18 NotificationOrchestrator | `case-action-events`, `core-request-events`, `external-request-events` | Pass-through of inbound `RECEIVED_KEY` — identity depends on COMP-16 upstream | Likely deviation. Cross-component pattern |
| COMP-19 AcceptService | `internal-integration-events` | `caseNumber` from `AddActionResponse`. `merchantId` exists on `CaseLookupResponse` but is never mapped into `AcceptEvent`. | ⚠️ DEC-003 deviation confirmed; no documented reason in source |
| COMP-23 CaseManagementService | `business-rules` | `caseNumber` | Confirmed — intentional for case-level ordering |
| COMP-24 CaseActionService | `business-rules`, ActionEvent topic | `caseNumber` | Confirmed — consistent with platform-wide deviation pattern across all five business-rules publishers |
| COMP-25 NotesService | `business-rules` | `caseNumber` | Confirmed from source |
| COMP-37 DocumentManagementService | `business-rules` | `caseNumber` on every reachable publish (legacy `merchantId` path is dead code, 0 callers) | Uniform deviation — dead-code legacy method `sendBusinessRules(merchantId)` exists but is never invoked |
| COMP-43 CoreNotificationConsumer | `core-request-events` (consumer side) | `caseNumber` (consumer-side variable name on `@Header(KafkaHeaders.RECEIVED_KEY)` parameter) | Producer-side key assignment confirmation owed by COMP-18 |
| COMP-41 ThirdPartyNotificationConsumer | `external-request-events` (consumer side) | Pass-through key — read as `@Header(KafkaHeaders.RECEIVED_KEY) String keyMerchantId`, only logged. Not used for routing. Producer-side partition key is COMP-18's responsibility (DEC-003 verifiability gap). | Consumer-side observation only |
| COMP-42 BENConsumer | `external-request-events` (consumer side) | Inbound key mapped to local variable `caseNumber` — strongly implies key is `caseNumber`, not `merchantId`. **Requires confirmation from COMP-18 producer source.** | Consumer-side observation |

> ℹ️ **business-rules topic — uniform DEC-003 deviation across all six publishers.** Every confirmed publisher (COMP-15, COMP-23, COMP-24, COMP-25, COMP-37, COMP-12 Scheduler4) uses `caseNumber`. The deviation is no longer "whether to deviate" — candidate ADR pending on whether to formally update DEC-003 to recognise case-scoped ordering on this topic, or undertake a platform-wide migration to `merchantId`.

---

## 3. Topic Registry

### How to read this table

- **Publisher:** The component(s) that write to this topic.
- **Consumer(s):** The component(s) that read from this topic.
- **Partition key:** The Kafka message key used for partition routing.
- **Payload:** Link to the component file that owns the payload schema.

---

| Topic Name | Publisher | Consumer(s) | Partition Key | Status | Payload Schema |
|------------|-----------|-------------|---------------|--------|----------------|
| `nap-dispute-events` | COMP-04 NAPDisputeEventService | COMP-05 NAPDisputeEventProcessor | merchantId (all events) / cardAcceptorCodeId (POST /event new disputes only) — see note [2] | ✅ Production | See WDP-COMP-04-NAP-DISPUTE-EVENT-SERVICE.md Block C |
| `new-case-events` (prod) / `new-case-events-cert` (cert) | COMP-12 InboundDisputeEventScheduler | COMP-14 CaseCreationConsumer | Compound key: `networkCaseId + cardNetwork + platform` — ⚠️ DEC-003 deviation, not merchantId. Consumer-side confirmation: key logged as `keyNetworkCaseCardNetworkId`, not used for routing inside consumer. | ✅ Production | See WDP-COMP-12-INBOUND-EVENT-SCHEDULER.md Block C + WDP-COMP-14-CASE-CREATION-CONSUMER.md Block B |
| `case-evidence-events` (prod) / `case-evidence-events-dev` / `-stg` / `-cert` | COMP-12 InboundDisputeEventScheduler | COMP-15 EvidenceConsumer | `caseNumber` if non-blank; else `networkCaseId` — ⚠️ DEC-003 deviation, not merchantId. **Inbound `RECEIVED_KEY` (caseNumber) is `@Nullable`** — ordering guarantee by `caseNumber` only holds when upstream sets a non-null key. | ✅ Production | See WDP-COMP-15-EVIDENCE-CONSUMER.md Block B |
| `business-rules` (prod) / `business-rules-cert` / `business-rules-dev` | COMP-12 InboundDisputeEventScheduler (Scheduler4, component=BUSINESS_RULES rows); COMP-15 EvidenceConsumer (WDP path, isMultiDocPending=false — DEC-001 deviation, inside `@Transactional` via `kafkaTemplate.send(...).get()`); COMP-23 CaseManagementService (after every material case write — DEC-001 deviation, kafka-before-commit pattern); COMP-24 CaseActionService (after action create/update — DEC-001 PARTIAL — BRE inside @Transactional); COMP-25 NotesService (non-SNOTE writes only — DEC-001 deviation); COMP-37 DocumentManagementService (caseNumber key — DEC-003 deviation — publishes on upload Endpoints 1, 9, 10 and questionnaire Endpoint 11, gated by `notifyBRQueue` + `startRuleGroup`; Endpoint 8 NAP base64 path never publishes; questionnaire path wraps publish in `@Transactional(rollbackOn=Exception.class)` — stronger atomicity than other publishers on this topic) ✅ COMP-14 candidacy resolved 2026-04-18 — **CaseCreationConsumer is NOT a publisher** (no `KafkaTemplate`, no `ProducerFactory`, no reference to `business-rules` topic in source). | COMP-16 BusinessRulesProcessor | `caseNumber` — ⚠️ DEC-003 deviation, not merchantId (uniform across all six publishers) | ✅ Production | See WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md Block B |
| `outgoing-events` (prod) / `outgoing-events-{dev/uat/stg/cert}` | COMP-16 BusinessRulesProcessor | COMP-18 NotificationOrchestrator | `caseNumber` — ⚠️ DEC-003 deviation. Published in `finally` block — always emitted if `caseDetails + currentActionDetails` non-null. ⚠️ DEC-001 deviation — no transactional outbox. | ✅ Production | See WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md Block C |
| `internal-integration-events` | COMP-19 AcceptService (caseNumber key — ⚠️ DEC-003 deviation); COMP-20 ContestService (merchantId key — DEC-003 compliant; up to **2** publishes per request when CRMR action code present — *corrected 2026-04-23 from previously documented 3* — branches are `if/else if`, mutually exclusive); COMP-16 BusinessRulesProcessor (conditional — UK/NAP path only, caseNumber key — ⚠️ DEC-003 deviation — DMT001-only publish is independent of the finally-block `outgoing-events` publish; both can fire for the same message) | COMP-39 NAPOutcomeProcessor (NAP platform filter only — silently discards non-NAP); COMP-40 VisaResponseQuestionnaire (non-null visaResponseIds filter only — silently discards AcceptService events) | Varies by publisher — caseNumber (COMP-19, COMP-16) or merchantId (COMP-20) | ✅ Production | See WDP-COMP-19-ACCEPT-SERVICE.md + WDP-COMP-20-CONTEST-SERVICE.md Block C |
| `case-action-events` (prod) / `case-action-events-cert` (cert) | COMP-18 NotificationOrchestrator (Filter 1 EXPIRY_EVENT routing) | COMP-17 CaseExpiryUpdateConsumer | Pass-through of `KafkaHeaders.RECEIVED_KEY` from upstream — ⚠️ DEC-003 likely deviation | ✅ Production | See WDP-COMP-17-CASE-EXPIRY-CONSUMER.md Block B + WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md Block C |
| `external-request-events` (prod) / `external-request-events-stg` / `-dev` | COMP-18 NotificationOrchestrator | COMP-41 ThirdPartyNotificationConsumer, COMP-42 BENConsumer, COMP-44 EDIAConsumer (🔴 Planned) | Pass-through of `KafkaHeaders.RECEIVED_KEY` — ⚠️ DEC-003 likely deviation. No secondary routing at publish time — downstream consumers apply their own filtering. COMP-41 reads key as `keyMerchantId` (logged only, not used). COMP-42 reads key into local variable `caseNumber` (suggests caseNumber). | ✅ Production | See WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md Block C |
| `core-request-events` | COMP-18 NotificationOrchestrator | COMP-43 CoreNotificationConsumer | Pass-through of `KafkaHeaders.RECEIVED_KEY` — ⚠️ DEC-003 deviation. Consumer-side confirmed: variable name `caseNumber` on `@Header(KafkaHeaders.RECEIVED_KEY)` parameter. Producer-side confirmation owed by COMP-18. | ✅ Production | See WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md Block C + WDP-COMP-43-CORE-NOTIFICATION-CONSUMER.md Block B |
| `${kafka.topic}` — ActionEvent (actual topic name TBC from env config) | COMP-24 CaseActionService (conditional — when `napUpdateEvent=true`) | ⚠️ TBC — likely COMP-39 NAPOutcomeProcessor based on platform role | caseNumber — ⚠️ DEC-003 deviation | ✅ Production | See WDP-COMP-24-CASE-ACTION-SERVICE.md Block C |
| `EDIA-events` | COMP-44 EDIAConsumer | EDIA Platform (external) | TBD | 🔴 Planned | ⚠️ PENDING — EDIA format TBD |

**Notes:**

[1] **business-rules topic publishers — finalised list (2026-04-23):** Six confirmed publishers — COMP-12 (Scheduler4 bre_orchestration_outbox rows), COMP-15 (WDP path, isMultiDocPending=false), COMP-23, COMP-24, COMP-25, COMP-37 (upload Endpoints 1/9/10 and questionnaire Endpoint 11). All use `caseNumber` as partition key — DEC-003 uniform deviation. **COMP-14 candidacy is resolved — NOT a publisher** (confirmed by absence audit 2026-04-18: zero `KafkaTemplate`, zero `ProducerFactory`, zero references to `business-rules` topic in `gcp-case-creation-consumer` source). This closes Observability-doc OQ-02.

[2] **nap-dispute-events partition key deviation from DEC-003:** The partition key for new dispute events (POST /event) is `cardAcceptorCodeId`, not `merchantId`. All other event types (case update, win/loss outcome) correctly use `merchantId`. Confirmed partial deviation. ⚠️ Decommission-scoped — this topic will be retired when the NAP/WPG inbound path is decommissioned.

[3] **AcceptEvent split-brain on `internal-integration-events` (2026-04-23):** AcceptEvent can be published in two scenarios where no card-network notification actually occurred: (a) MC CHI on NAP, (b) AMEX/DISCOVER on NAP with eligible inbound actionCode. NAPOutcomeProcessor consumers should NOT assume that an AcceptEvent implies the network was successfully notified. ADR pending.

[4] **MCM stage-dependent path on COMP-20 ContestService:** MCM submission is stage-dependent — CH1/RE2 uses `createSecondPresentmentChargeback` (POST); PAB uses `retrieveClaim + updateCase(action=REJECT)` (GET then PUT). The DataPower non-NAP path uses the same VANTIV license header as Visa DataPower (shared `licenseKey` secret).

[5] **`channelTypeTopicMap` non-prod default keys (COMP-12, corrected 2026-04-18):** `EXPIRY_EVENTS`, `CORE_EVENTS`, `GP_EVENTS`, `BEN_EVENTS` — *corrected from previously documented `SEN_EVENTS`*. Source: `application-local.yml:14`. Production values remain in K8s secrets. ⚠️ Note: COMP-41 channel_type was likewise corrected from `GF_EVENTS` to `GP_EVENTS` (2026-04-25) — see WDP-DB.md `wdp.outgoing_event_outbox` row.

[6] **Kafka-metadata write-back asymmetry (COMP-12, 2026-04-18):** `new-case-events` and `case-evidence-events` paths persist `kafka_offset` / `kafka_partition` / `kafka_topic` back to the source outbox row. The other three paths (Scheduler3 → `outgoing_event_outbox`; Scheduler4 → `bre_orchestration_outbox`) pass a NULL entity and **do not persist Kafka metadata** to those tables. Incident correlation between Kafka logs and outbox rows on those paths requires log-side join on `idempotencyId`.

---

## 4. Consumer Group Registry

Consumer group IDs are injected via Kubernetes secrets in most components
and are not committed to source repositories. This table captures group IDs
as they are confirmed from component analysis or environment inspection.

| Consumer Group ID | Topic | Component | Offset Strategy | Confirmed |
|-------------------|-------|-----------|-----------------|-----------|
| `${kafka_group_id}` — value injected via K8s secret | `nap-dispute-events` | COMP-05 NAPDisputeEventProcessor | Pre-ACK — `acknowledgment.acknowledge()` before all processing (DEC-005 deviation) | ⚠️ Group ID value not in source |
| `new-case-events-group` (prod) / `new-case-events-group-cert` (cert) | `new-case-events` | COMP-14 CaseCreationConsumer | Pre-ACK — `MANUAL_IMMEDIATE` with `syncCommits=true`, ack at `KafkaConsumer.java:38` before `processKafkaNotificationEvent()` at `:43` (DEC-005 deviation). Concurrency: 1 (default — `setConcurrency()` never called). `auto.offset.reset = latest` — cold start with no committed offset **skips** messages, not replay. `enable.auto.commit=false`, `allow.auto.create.topics=false`. Empty anonymous `CommonErrorHandler{}` — deserialization exceptions silently swallowed; no DLT, no log, no counter. `${max_poll_records}`, `${max_poll_interval}`, `${session_timeout_ms}`, `${heartbeat_interval_ms}` — all env-injected, no YAML defaults. Key deserializer `StringDeserializer`; value deserializer `ErrorHandlingDeserializer` wrapping `JsonDeserializer<NotificationEvent>`. SASL_SSL + AWS_MSK_IAM. *(2026-04-18: cert profile naming corrected from previously documented `-dev`.)* | ✅ Confirmed from COMP-14 source |
| `case-evidence-events-group` (prod) / `case-evidence-events-group-cert` (cert) | `case-evidence-events` | COMP-15 EvidenceConsumer | Pre-ACK — `MANUAL_IMMEDIATE` with `syncCommits=true` at Step 1, before all processing (DEC-005 deviation). Concurrency: 1 (default). Max poll records: **500**. Max poll interval: **600 000 ms** (10 minutes). `auto.offset.reset=latest`, `enable.auto.commit=false`, `allow.auto.create.topics=false`. SASL_SSL + AWS_MSK_IAM. `idempotency-key` header is `@Nullable`. Bad-payload behaviour: `ErrorHandlingDeserializer` produces null; empty `CommonErrorHandler` causes NPE in listener; caught and logged; no DB write; message permanently lost. | ✅ Confirmed from COMP-15 source |
| `business-rules-processor-group` (prod) / `business-rules-group` (dev/local) | `business-rules` | COMP-16 BusinessRulesProcessor | Pre-ACK — `MANUAL_IMMEDIATE` committed on `KafkaConsumer.java:38`, before `processRulesEvent()` at `:41` (DEC-005 deviation). Concurrency: 1 (Spring default — `setConcurrency()` not invoked). `auto.offset.reset=latest`, `enable.auto.commit=false`, `allow.auto.create.topics=false`. `${max_poll_records}` / `${session_timeout_ms}` / `${heartbeat_interval_ms}` env-injected. Empty anonymous `CommonErrorHandler{}` — silent drop. SASL_SSL + AWS_MSK_IAM. Inbound headers: `RECEIVED_KEY → keyMerchantId` (logged only); `idempotency-key → idempotencyId` (passed through, NOT used for dedup). | ✅ Confirmed from COMP-16 source |
| `outgoing-events-group` (prod) | `outgoing-events` | COMP-18 NotificationOrchestrator | Mid-flow ACK — `MANUAL_IMMEDIATE` with `syncCommits=true`, single ACK call site committed AFTER outbox INSERT (Step 3d) but BEFORE all Step 7 publishes (DEC-005 deviation). Concurrency: 1 (default). `${max_poll_records}` / `${max_poll_interval}` / `${session_timeout_ms}` / `${heartbeat_interval_ms}` env-injected, no YAML defaults. `auto.offset.reset=latest`, `auto.commit=false`. Value deserializer `ErrorHandlingDeserializer<NotificationEvent>` wrapping `JsonDeserializer`. Factory-level no-op `CommonErrorHandler` — silent drop on bad payload. SASL_SSL + AWS_MSK_IAM. | ✅ Confirmed from COMP-18 source |
| `case-action-events-group` (prod) / `case-action-events-group-cert` (cert) | `case-action-events` | COMP-17 CaseExpiryUpdateConsumer | `MANUAL_IMMEDIATE` + `syncCommits=true` — ⚠️ ACK timing inconsistent: Path A pre-ACK before all writes; Path B mid-flow ACK after outbox INSERT but before `case_expiry` write; header-blank path post-error-row INSERT. DEC-005 deviation. Concurrency: 1 (Spring default). **Max poll records: 500** *(corrected 2026-04-25 from previously documented 280)*. **Max poll interval: 600 000 ms (10 min)** *(corrected 2026-04-25 from previously documented `300000`)*. `auto.offset.reset=latest`, `enable.auto.commit=false`, `allow.auto.create.topics=false`. SASL/SSL: `AWS_MSK_IAM` over `SASL_SSL` (added 2026-04-25). Wire-level Kafka headers are kebab-case: `idempotency-key`, `event-timestamp` *(corrected 2026-04-25 from camelCase)*. Bad-payload behaviour: deserialisation failure → NPE at path-selection branch, **no ACK in either path**; same poison message redelivers indefinitely until `max.poll.interval.ms` (10 min) trips a rebalance. | ✅ Confirmed from COMP-17 source |
| ⚠️ PENDING — value injected via K8s secret | `internal-integration-events` *(confirmed runtime value 2026-04-23 — was previously inferred)* | COMP-39 NAPOutcomeProcessor | Pre-ACK — `acknowledgment.acknowledge()` as first call in `onMessage()`, before all processing (DEC-005 deviation). Concurrency: 1 (Spring Kafka default — `setConcurrency()` not invoked). `${max_poll_records}` / `${max_poll_interval}` env-injected. SASL_SSL + AWS_MSK_IAM. Filters by `platform` field in payload (NAP only). | ✅ Pattern confirmed from COMP-39 source; group ID value TBC |
| ⚠️ PENDING — value injected via K8s secret | `internal-integration-events` | COMP-40 VisaResponseQuestionnaire | Pre-ACK (consistent with platform-wide pattern — confirmed from COMP-40 source). Filters by non-null `visaResponseIds` (silently discards AcceptService events). | ✅ Pattern confirmed; group ID value TBC |
| `${kafka_group_id}` — env-injected (`spring.kafka.consumer.groupId`) | `external-request-events` (env-injected `${kafka_topic}`) | COMP-41 ThirdPartyNotificationConsumer | Pre-Signifyd ACK — `MANUAL_IMMEDIATE` + `syncCommits=true`. Committed after PUBLISHED outbox INSERT, before `processEvent()` and any Signifyd REST call (DEC-005 deviation). Concurrency: 1 (default — no `setConcurrency()`). `${max_poll_records}`, `${max_poll_interval}` env-injected. `auto.offset.reset=latest` — cold-start backlog skipped. `enable.auto.commit=false`, `allow.auto.create.topics=false`. SASL_SSL + AWS_MSK_IAM. Bad-payload behaviour: empty `CommonErrorHandler` silently drops bad-deserialise records — no DLT, no halt, no audit row. Key deserialiser `StringDeserializer`; value deserialiser `ErrorHandlingDeserializer` wrapping `JsonDeserializer<BusinessRuleEvent>`. | ✅ Pattern confirmed from COMP-41 source |
| `external-request-ben-events` (prod) / `external-request-ben-events-stg` / `external-request-ben-events-dev` | `external-request-events` (prod) / `external-request-events-stg` / `external-request-events-dev` | COMP-42 BENConsumer | Pre-ACK — `MANUAL_IMMEDIATE` + `syncCommits=true`. Committed after idempotency DB check (Step 3); BEFORE CMS call (Step 6), BEN Product call (Step 7), CAS call (Step 10), BEN Kafka publish (Step 12), outbox status update (Step 13). DEC-005 deviation. Concurrency: 1 (default). `${max_poll_records}` / `${max_poll_interval}` env-injected, value not in source. `auto.offset.reset=latest`, `allow.auto.create.topics=false`. SASL_SSL + AWS_MSK_IAM. Deserializer `ErrorHandlingDeserializer<BusinessRuleEvent>` wrapping `JsonDeserializer`. Bad payload: ErrorHandlingDeserializer null + empty CommonErrorHandler — silently dropped, no logging, no dead-letter write. ⚠️ Observability gap. | ✅ Confirmed from COMP-42 source |
| `core-request-events-group` | `core-request-events` | COMP-43 CoreNotificationConsumer | Pre-ACK on all paths — `MANUAL_IMMEDIATE` + `syncCommits=true`. Path A: post outbox-ERROR write, pre-DB2; Path B: pre `processCaseActionEvent`; Path C: post outbox-PUBLISHED INSERT, pre-DB2 (DEC-005 deviation). Concurrency: 1 (Spring default). `max.poll.records=500`, `max.poll.interval.ms=600000` (10 min). `session.timeout.ms` and `heartbeat.interval.ms` not configured — Apache Kafka defaults (45 000 / 3 000). `auto.offset.reset=latest`, `enable.auto.commit=false`, `allow.auto.create.topics=false`. SASL_SSL + AWS_MSK_IAM. Empty anonymous `CommonErrorHandler{}` — bad payload causes NPE at first event-field dereference; outer try/catch logs only, NO ACK, no outbox write; redelivers up to `max.poll.interval.ms` then rebalances. **Persistent bad payloads can produce a rebalance loop.** | ✅ Confirmed from COMP-43 source |
| ⚠️ PENDING | `external-request-events` | COMP-44 EDIAConsumer | ⚠️ TBC | 🔴 Planned — not yet built |

---

## 5. Outbox Tables That Feed Kafka

WDP uses the transactional outbox pattern (DEC-001) extensively. These
tables are written by processing components and read by schedulers to
relay events to Kafka.

| Outbox Table | Written by | Read / Published by | Kafka Topic Target | Notes |
|--------------|------------|---------------------|--------------------|----|
| `wdp.chbk_outbox_row` | COMP-07 VisaDisputeBatch (`WVDPB`), COMP-08 FirstChargebackBatch (`WMFDPB`), COMP-09 CaseFillingBatch (`WMPAPB`), COMP-11 FileProcessor (`FILE_PROCESSOR` — *corrected 2026-04-18 from `WPFLEPR`*) — INSERT PENDING/SKIPPED rows; COMP-12 InboundDisputeEventScheduler (status transitions + Kafka metadata write-back on `new-case-events` and `case-evidence-events` paths only); COMP-14, COMP-15, COMP-23, COMP-24 (status updates) | COMP-12 InboundDisputeEventScheduler (Scheduler1 — reads PENDING rows) | `new-case-events`, `case-evidence-events` | Primary inbound outbox. `idempotencyId` typed as **UUID** — cross-table inconsistency with `outgoing_event_outbox` (String). |
| `wdp.outgoing_event_outbox` | COMP-17 CaseExpiryUpdateConsumer (channel_type=EXPIRY_EVENTS, `WCSEEXPC`); COMP-41 ThirdPartyNotificationConsumer (channel_type=GP_EVENTS, `WNEC` — *added 2026-04-25; channel_type corrected from previously documented `GF_EVENTS`*); COMP-43 CoreNotificationConsumer (channel_type=CORE_EVENTS, `PCSECRTC`) ⚠️ **COMP-18 REMOVED 2026-04-18** — was incorrectly documented as a writer; source grep confirmed zero references in `wdp-outgoing-consumer`. | COMP-12 InboundDisputeEventScheduler (Scheduler3 — reads FAILED and PENDING_DEFERRED rows only) | `case-action-events`, `core-request-events`, `external-request-events` (via `channelTypeTopicMap`) | ⚠️ Scheduler3 queries FAILED/PENDING_DEFERRED only — not PENDING, not PUBLISHED. PUBLISHED-orphan paths in COMP-41 (3 distinct paths) and COMP-43 are not auto-recovered by Scheduler3. `idempotencyId` typed as **String** — cross-table inconsistency with `chbk_outbox_row` (UUID). Scheduler3 channel_type filter behaviour is OQ-COMP41-1. |
| `wdp.bre_orchestration_outbox` | COMP-18 NotificationOrchestrator (component=NOTIFICATION_ORCHESTRATOR rows, `WDPOUCU` — four distinct write points, no `@Transactional`); COMP-12 InboundDisputeEventScheduler (component=BUSINESS_RULES rows) | COMP-12 InboundDisputeEventScheduler (Scheduler4 — reads FAILED and PENDING_DEFERRED rows only) | `business-rules` (component=BUSINESS_RULES rows), `outgoing-events` *(refer COMP-18 publish targets)* | ⚠️ Scheduler4 queries FAILED/PENDING_DEFERRED only — not PUBLISHED. PUBLISHED-status orphan rows have no automatic re-drive path — **manual runbook required for RISK-015**. |
| `wdp.file_evidence` | COMP-11 FileProcessor (`FILE_PROCESSOR`) | COMP-12 InboundDisputeEventScheduler (Scheduler5 — error report only, read-only) | N/A — evidence tracking only | Updated by COMP-15 EvidenceConsumer (status transitions) |
| `wdp.case_expiry` | COMP-17 CaseExpiryUpdateConsumer | Downstream expiry-driven workflows (TBC) | N/A — schedule tracking, not Kafka relay | ⚠️ Downstream consumers of this table not yet confirmed. UPSERT is application-level (no DB-level INSERT … ON CONFLICT). |
| `wdp.file_generation_event` | COMP-18 NotificationOrchestrator (sole writer, `WDPOUCU` — INSERT-only, status hardcoded `STAGED`) | COMP-45, COMP-46, COMP-47 (file-based output processors — not yet documented) | N/A — staging table for file outputs | ⚠️ Previously documented as `wdp.file_notifications` — that table name does not exist in the codebase; correction confirmed by COMP-18 v2.0 audit. |

---

## 6. Components Confirmed Kafka-Free

No Kafka producer, consumer, or dependency of any kind.

| Component | Confirmation basis |
|-----------|-------------------|
| COMP-01 API Gateway | No Kafka dependency in pom.xml. No KafkaTemplate or @KafkaListener. Confirmed from source. |
| COMP-02 UAMS | No Kafka dependency in pom.xml. No KafkaTemplate or @KafkaListener. Confirmed from source. |
| COMP-03 CHAS | No spring-kafka dependency in pom.xml. Confirmed from source. |
| COMP-07 VisaDisputeBatch | Kafka dependencies staged (POM only) but zero producers or consumers wired. Confirmed Kafka-free 2026-04-18 — grep against source for `@EnableKafka`, `KafkaTemplate`, `ProducerFactory`, `@KafkaListener` returns zero hits. |
| COMP-08 FirstChargebackBatch | Kafka dependencies staged (`spring-kafka`, `aws-msk-iam-auth`) but zero producers or consumers wired. Confirmed Kafka-free 2026-04-18. |
| COMP-09 CaseFillingBatch | Consistent with COMP-07/08 pattern — zero producers or consumers wired. Confirmed Kafka-free 2026-04-18. |
| COMP-11 FileProcessor | SQS-triggered — uses Spring Cloud AWS SQS listener, not Kafka. Zero `spring-kafka`, `kafka-clients`, `spring-cloud-stream` in pom.xml. Confirmed Kafka-free 2026-04-18. |
| COMP-13 FileAcknowledgementProcessor | Confirmed Kafka-free 2026-04-18 — zero `spring-kafka`, `kafka-clients`, `aws-msk-iam-auth` in `pom.xml` and zero `@KafkaListener` / `KafkaTemplate` in source. |
| COMP-14 CaseCreationConsumer | Consumer of `new-case-events` only. Confirmed 2026-04-18 to have **no Kafka producer side** — no `KafkaTemplate`, no `ProducerFactory`, no reference to `business-rules` topic. (Closes Observability-doc OQ-02.) |
| COMP-21 ChargebackService | REST API only. No Kafka dependency in pom.xml. Confirmed Kafka-free 2026-04-25 by full-tree grep — zero `@KafkaListener`, `KafkaTemplate`, `@EnableKafka`, `kafka.bootstrap` occurrences. |
| COMP-22 DisputeService | Kafka producer wired to `business-rules` but call site commented out — effectively Kafka-free at runtime. Confirmed from source. |
| COMP-27 CaseSearchService | REST API only. No Kafka dependency. Confirmed Kafka-free 2026-04-24 by full-tree grep — zero `@KafkaListener`, `KafkaTemplate`, `@EnableKafka`, no `spring-kafka` in pom.xml. |
| COMP-28 DisplayCodeService | REST API only. No Kafka dependency. Confirmed from source. |
| COMP-29 FaxQueueService | REST API only. No Kafka dependency. Confirmed from source. |
| COMP-30 UserQueueSkillService | REST API only. No Kafka dependency. Confirmed from source. |
| COMP-31 BusinessRulesService | No Kafka dependency in pom.xml. Confirmed from source. |
| COMP-32 RulesService | No Kafka dependency in pom.xml. Confirmed from source. |
| COMP-34 MerchantTransactionService | REST API read-only. No Kafka dependency. Confirmed from source. |
| COMP-35 EncryptionService | REST API. No Kafka dependency. Confirmed from source. |
| COMP-36 TokenService | REST API. No Kafka dependency. Confirmed from source. |
| COMP-38 APILogService | REST API sink. No Kafka dependency. Confirmed from source. |

> 🔴 **REMOVED from this list 2026-04-23: COMP-37 DocumentManagementService.** Previously documented as Kafka-free; source-verification has confirmed it IS a publisher of `business-rules` (caseNumber key, DEC-003 deviation) on Endpoints 1, 9, 10, and 11. See Section 3 Topic Registry.

---

*Last updated: 2026-04-25*
*Version 2.1 — reconciled from WDP-CHANGE-LOG.md Pending Entries*
*Next reconciliation: when next batch of architect-confirmed component files lands.*
