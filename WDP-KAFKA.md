# WDP-KAFKA.md
**Worldpay Dispute Platform — Kafka Topic Registry**
*Version: 2.0 | April 2026*
*Updated: All component files uploaded and verified April 2026*

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
| Authentication | AWS IAM (MSK IAM auth — SASL_SSL + IAMLoginModule) | Confirmed from component analysis |
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
| Idempotency key | UUID carried as Kafka message header `idempotency-key` | Platform pattern |
| Message ordering | Guaranteed within partition (per merchantId) | DEC-003 |
| Producer idempotence | enable.idempotence=true on all Kafka producers | Per component |

---

### Confirmed DEC-005 Deviations (offset commit timing)

Every confirmed Kafka consumer in WDP uses pre-ACK or mid-flow ACK
rather than the DEC-005 standard of committing after full processing.
This is a platform-wide pattern, not an isolated exception.

| Component | Topic | Deviation detail |
|-----------|-------|-----------------|
| COMP-05 NAPDisputeEventProcessor | nap-dispute-events | Pre-ACK — first call in onMessage() |
| COMP-12 InboundDisputeEventScheduler | (producer only — outbox mark-before-send) | Mark-before-send — PUBLISHED status written before Kafka send |
| COMP-14 CaseCreationConsumer | new-case-events | Pre-ACK — line 36 of KafkaConsumer.java before processKafkaNotificationEvent() |
| COMP-15 EvidenceConsumer | case-evidence-events | Pre-ACK — Step 1, before all processing. syncCommits=true |
| COMP-16 BusinessRulesProcessor | business-rules | Pre-ACK — line 37 of KafkaConsumer.java before processRulesEvent() |
| COMP-17 CaseExpiryUpdateConsumer | case-action-events | Inconsistent — Path A pre-ACK; Path B mid-flow ACK after outbox INSERT |
| COMP-18 NotificationOrchestrator | outgoing-events | Mid-flow ACK — after outbox INSERT (Step 3d) but BEFORE Kafka publishes (Step 7) |
| COMP-39 NAPOutcomeProcessor | internal-integration-events | Pre-ACK — acknowledgment.acknowledge() as first call in onMessage() |
| COMP-40 VisaResponseQuestionnaire | internal-integration-events | Pre-ACK (consistent with platform pattern — confirmed from COMP-40) |

### Confirmed DEC-001 Deviations (transactional outbox)

| Component | Topic | Deviation detail |
|-----------|-------|-----------------|
| COMP-04 NAPDisputeEventService | nap-dispute-events | Direct publish on HTTP thread — no outbox |
| COMP-15 EvidenceConsumer | business-rules | Synchronous publish inside @Transactional — ghost event possible on rollback |
| COMP-16 BusinessRulesProcessor | outgoing-events, internal-integration-events | Direct synchronous publish — no outbox. Broker unavailable = permanent event loss |
| COMP-19 AcceptService | internal-integration-events | Direct synchronous publish — no outbox |
| COMP-20 ContestService | internal-integration-events | Direct synchronous publish — no outbox |
| COMP-23 CaseManagementService | business-rules | Synchronous publish after DB transaction commits — Kafka failure post-commit is unrecoverable |
| COMP-24 CaseActionService | business-rules, ActionEvent topic | Synchronous publish post-DB-commit — split-brain risk confirmed |
| COMP-25 NotesService | business-rules | Synchronous publish inside @Transactional boundary — not atomically coupled |

### Confirmed DEC-003 Deviations (merchantId partition key)

| Component | Topic | Actual key | Note |
|-----------|-------|-----------|------|
| COMP-04 NAPDisputeEventService | nap-dispute-events | cardAcceptorCodeId (POST /event only); merchantId for other event types | Partial deviation — decommission-scoped |
| COMP-12 InboundDisputeEventScheduler | new-case-events | Compound: networkCaseId + cardNetwork + platform | Confirmed from source |
| COMP-12 InboundDisputeEventScheduler | case-evidence-events | caseNumber if non-blank; else networkCaseId | Confirmed from source |
| COMP-15 EvidenceConsumer | business-rules (producer) | caseNumber | Confirmed from source |
| COMP-16 BusinessRulesProcessor | outgoing-events, internal-integration-events | caseNumber | Confirmed from source |
| COMP-18 NotificationOrchestrator | case-action-events, core-request-events, external-request-events | Pass-through of inbound RECEIVED_KEY — identity depends on COMP-16 upstream | Likely deviation |
| COMP-19 AcceptService | internal-integration-events | caseNumber | ⚠️ DEC-003 deviation confirmed |
| COMP-23 CaseManagementService | business-rules | caseNumber | Confirmed — intentional for case-level ordering |
| COMP-24 CaseActionService | business-rules, ActionEvent topic | caseNumber | Confirmed from source |
| COMP-25 NotesService | business-rules | caseNumber | Confirmed from source |

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
| `new-case-events` | COMP-12 InboundDisputeEventScheduler | COMP-14 CaseCreationConsumer | Compound key: networkCaseId + cardNetwork + platform — ⚠️ DEC-003 deviation, not merchantId | ✅ Production | See WDP-COMP-12-INBOUND-EVENT-SCHEDULER.md Block C + WDP-COMP-14-CASE-CREATION-CONSUMER.md Block B |
| `case-evidence-events` | COMP-12 InboundDisputeEventScheduler | COMP-15 EvidenceConsumer | caseNumber if non-blank; else networkCaseId — ⚠️ DEC-003 deviation, not merchantId | ✅ Production | See WDP-COMP-15-EVIDENCE-CONSUMER.md Block B |
| `business-rules` | COMP-12 InboundDisputeEventScheduler (Scheduler4, component=BUSINESS_RULES rows); COMP-15 EvidenceConsumer (WDP path, isMultiDocPending=false — DEC-001 deviation, inside @Transactional); COMP-23 CaseManagementService (after every material case write — DEC-001 deviation); COMP-24 CaseActionService (after action create/update — DEC-001 deviation); COMP-25 NotesService (non-SNOTE writes only — DEC-001 deviation) ⚠️ COMP-14 CaseCreationConsumer candidacy still unverified — see note [1] | COMP-16 BusinessRulesProcessor | caseNumber — ⚠️ DEC-003 deviation, not merchantId (confirmed across all publishers) | ✅ Production | See WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md Block B |
| `outgoing-events` | COMP-16 BusinessRulesProcessor | COMP-18 NotificationOrchestrator | caseNumber — ⚠️ DEC-003 deviation. Published in finally block — always emitted if caseDetails + currentActionDetails non-null. ⚠️ DEC-001 deviation — no transactional outbox. | ✅ Production | See WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md Block C |
| `internal-integration-events` | COMP-19 AcceptService (caseNumber key — ⚠️ DEC-003 deviation); COMP-20 ContestService (merchantId key — DEC-003 compliant); COMP-16 BusinessRulesProcessor (conditional — UK/NAP path only, caseNumber key — ⚠️ DEC-003 deviation) | COMP-39 NAPOutcomeProcessor (NAP platform filter only — silently discards non-NAP); COMP-40 VisaResponseQuestionnaire (non-null visaResponseIds filter only — silently discards AcceptService events) | Varies by publisher — caseNumber (COMP-19, COMP-16) or merchantId (COMP-20) | ✅ Production | See WDP-COMP-19-ACCEPT-SERVICE.md + WDP-COMP-20-CONTEST-SERVICE.md Block C |
| `case-action-events` (expiry) | COMP-18 NotificationOrchestrator (confirmed — Filter 1 EXPIRY_EVENT routing) | COMP-17 CaseExpiryUpdateConsumer | Pass-through of `KafkaHeaders.RECEIVED_KEY` from upstream — ⚠️ DEC-003 likely deviation | ✅ Production | See WDP-COMP-17-CASE-EXPIRY-CONSUMER.md Block B + WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md Block C |
| `external-request-events` | COMP-18 NotificationOrchestrator | COMP-41 ThirdPartyNotificationConsumer, COMP-42 BENConsumer, COMP-44 EDIAConsumer (🔴 Planned) | Pass-through of `KafkaHeaders.RECEIVED_KEY` — ⚠️ DEC-003 likely deviation. No secondary routing at publish time — downstream consumers apply their own filtering. | ✅ Production | See WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md Block C |
| `core-request-events` | COMP-18 NotificationOrchestrator | COMP-43 CoreNotificationConsumer | Pass-through of `KafkaHeaders.RECEIVED_KEY` — ⚠️ DEC-003 likely deviation | ✅ Production | See WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md Block C + WDP-COMP-43-CORE-NOTIFICATION-CONSUMER.md Block B |
| `${kafka.topic}` — ActionEvent (actual topic name TBC from env config) | COMP-24 CaseActionService (conditional — when napUpdateEvent=true) | ⚠️ TBC — likely COMP-39 NAPOutcomeProcessor based on platform role | caseNumber — ⚠️ DEC-003 deviation | ✅ Production | See WDP-COMP-24-CASE-ACTION-SERVICE.md Block C |
| `EDIA-events` | COMP-44 EDIAConsumer | EDIA Platform (external) | TBD | 🔴 Planned | ⚠️ PENDING — EDIA format TBD |

**Notes:**

[1] **business-rules topic publisher — status:** Five publishers confirmed:
COMP-12 InboundDisputeEventScheduler (Scheduler4 bre_orchestration_outbox
rows), COMP-15 EvidenceConsumer (WDP path, isMultiDocPending=false),
COMP-23 CaseManagementService (after every material case write),
COMP-24 CaseActionService (after action create/update), COMP-25 NotesService
(non-SNOTE writes only). All use caseNumber as partition key — DEC-003
deviation confirmed across all. COMP-14 CaseCreationConsumer candidacy
still unverified — Copilot follow-up required on COMP-14 repo before
this open question is fully closed.

[2] **nap-dispute-events partition key deviation from DEC-003:**
The partition key for new dispute events (POST /event) is
cardAcceptorCodeId, not merchantId. All other event types (case
update, win/loss outcome) correctly use merchantId. This is a
confirmed partial deviation from DEC-003. Whether deliberate or
accidental is undocumented. ⚠️ Decommission-scoped — this topic will
be retired when the NAP/WPG inbound path is decommissioned.

---

## 4. Consumer Group Registry

Consumer group IDs are injected via Kubernetes secrets in most components
and are not committed to source repositories. This table captures group IDs
as they are confirmed from component analysis or environment inspection.

| Consumer Group ID | Topic | Component | Offset Strategy | Confirmed |
|-------------------|-------|-----------|-----------------|-----------|
| `${kafka_group_id}` — value injected via K8s secret | `nap-dispute-events` | COMP-05 NAPDisputeEventProcessor | Pre-ACK — acknowledgment.acknowledge() before all processing (DEC-005 deviation) | ⚠️ Group ID value not in source |
| `new-case-events-group` (prod) / `new-case-events-group-dev` (dev) | `new-case-events` | COMP-14 CaseCreationConsumer | Pre-ACK — MANUAL_IMMEDIATE before processKafkaNotificationEvent() (DEC-005 deviation). Concurrency: 1. | ✅ Confirmed from COMP-14 source |
| `case-evidence-events-group` (prod) | `case-evidence-events` | COMP-15 EvidenceConsumer | Pre-ACK — MANUAL_IMMEDIATE with syncCommits=true at Step 1, before all processing (DEC-005 deviation). Concurrency: 1. | ✅ Confirmed from COMP-15 source |
| `business-rules-processor-group` | `business-rules` | COMP-16 BusinessRulesProcessor | Pre-ACK — MANUAL_IMMEDIATE committed before processRulesEvent() (DEC-005 deviation). Concurrency: 1 (default). | ✅ Confirmed from COMP-16 source |
| `outgoing-events-group` (prod) | `outgoing-events` | COMP-18 NotificationOrchestrator | Mid-flow ACK — MANUAL_IMMEDIATE with syncCommits=true, committed AFTER outbox INSERT (Step 3d) but BEFORE Kafka publishes (Step 7). DEC-005 deviation. Concurrency: 1 (default). | ✅ Confirmed from COMP-18 source |
| `case-action-events-group` | `case-action-events` | COMP-17 CaseExpiryUpdateConsumer | MANUAL_IMMEDIATE, syncCommits=true — ⚠️ ACK timing inconsistent: Path A pre-ACK before all writes; Path B mid-flow ACK after outbox INSERT but before case_expiry write. DEC-005 deviation. Concurrency: 1. Max poll: 280. | ✅ Confirmed from COMP-17 source |
| ⚠️ PENDING — value injected via K8s secret | `internal-integration-events` | COMP-39 NAPOutcomeProcessor | Pre-ACK — acknowledgment.acknowledge() as first call in onMessage(), before all processing (DEC-005 deviation). Concurrency confirmed from source. | ✅ Pattern confirmed from COMP-39 source; group ID value TBC |
| ⚠️ PENDING — value injected via K8s secret | `internal-integration-events` | COMP-40 VisaResponseQuestionnaire | Pre-ACK (consistent with platform-wide pattern — confirmed from COMP-40 source) | ✅ Pattern confirmed; group ID value TBC |
| ⚠️ PENDING — value injected via K8s secret | `external-request-events` | COMP-41 ThirdPartyNotificationConsumer | ⚠️ TBC — confirm from COMP-41 source | ✅ File exists; group ID and strategy TBC |
| ⚠️ PENDING — value injected via K8s secret | `external-request-events` | COMP-42 BENConsumer | ⚠️ TBC — confirm from COMP-42 source | ✅ File exists; group ID and strategy TBC |
| ⚠️ PENDING — value injected via K8s secret | `core-request-events` | COMP-43 CoreNotificationConsumer | ⚠️ TBC — outbox pattern (wdp.outgoing_event_outbox channel_type=CORE_EVENTS) suggests at-most-once. Confirm from COMP-43 source. | ✅ File exists; group ID and strategy TBC |
| ⚠️ PENDING | `external-request-events` | COMP-44 EDIAConsumer | ⚠️ TBC | 🔴 Planned — not yet built |

---

## 5. Outbox Tables That Feed Kafka

WDP uses the transactional outbox pattern (DEC-001) extensively. These
tables are written by processing components and read by schedulers to
relay events to Kafka.

| Outbox Table | Written by | Read / Published by | Kafka Topic Target | Notes |
|--------------|------------|---------------------|--------------------|----|
| `wdp.chbk_outbox_row` | COMP-07 VisaDisputeBatch, COMP-08 FirstChargebackBatch, COMP-09 CaseFillingBatch, COMP-11 FileProcessor (PENDING rows); COMP-12 InboundDisputeEventScheduler (status transitions) | COMP-12 InboundDisputeEventScheduler (Scheduler1 — reads PENDING rows) | `new-case-events`, `case-evidence-events` | Primary inbound outbox |
| `wdp.outgoing_event_outbox` | COMP-17 CaseExpiryUpdateConsumer (channel_type=EXPIRY_EVENTS); COMP-18 NotificationOrchestrator (channel_type=notification orchestration rows); COMP-43 CoreNotificationConsumer (channel_type=CORE_EVENTS) | COMP-12 InboundDisputeEventScheduler (Scheduler3 — reads FAILED and PENDING_DEFERRED rows only) | `case-action-events`, `core-request-events`, `external-request-events` (via channelTypeTopicMap) | ⚠️ Scheduler3 queries FAILED/PENDING_DEFERRED only — not PENDING |
| `wdp.bre_orchestration_outbox` | COMP-18 NotificationOrchestrator (component=NOTIFICATION_ORCHESTRATOR rows); COMP-12 InboundDisputeEventScheduler (component=BUSINESS_RULES rows) | COMP-12 InboundDisputeEventScheduler (Scheduler4 — reads FAILED and PENDING_DEFERRED rows only) | `business-rules` (component=BUSINESS_RULES rows), `outgoing-events` (component=NOTIFICATION_ORCHESTRATOR rows) | ⚠️ Scheduler4 queries FAILED/PENDING_DEFERRED only — not PUBLISHED. PUBLISHED-status orphan rows have no automatic re-drive path. |
| `wdp.file_evidence` | COMP-11 FileProcessor | COMP-12 InboundDisputeEventScheduler (Scheduler5 — error report only, read-only) | N/A — evidence tracking only | |
| `wdp.case_expiry` | COMP-17 CaseExpiryUpdateConsumer | Downstream expiry-driven workflows (TBC) | N/A — schedule tracking, not Kafka relay | ⚠️ Downstream consumers of this table not yet confirmed |

---

## 6. Components Confirmed Kafka-Free

No Kafka producer, consumer, or dependency of any kind.

| Component | Confirmation basis |
|-----------|-------------------|
| COMP-01 API Gateway | No Kafka dependency in pom.xml. No KafkaTemplate or @KafkaListener. Confirmed from source. |
| COMP-02 UAMS | No Kafka dependency in pom.xml. No KafkaTemplate or @KafkaListener. Confirmed from source. |
| COMP-03 CHAS | No spring-kafka dependency in pom.xml. Confirmed from source. |
| COMP-08 FirstChargebackBatch | Kafka dependencies staged (spring-kafka, aws-msk-iam-auth) but zero producers or consumers wired. Confirmed Kafka-free from source. |
| COMP-09 CaseFillingBatch | Consistent with COMP-07/08 pattern — zero producers or consumers wired. Confirmed Kafka-free from source. |
| COMP-11 FileProcessor | SQS-triggered — uses Spring Cloud AWS SQS listener, not Kafka. Zero Kafka infrastructure. Confirmed from source. |
| COMP-21 ChargebackService | REST API only. No Kafka dependency in pom.xml. Confirmed from source. |
| COMP-22 DisputeService | Kafka producer wired to business-rules but call site commented out — effectively Kafka-free at runtime. Confirmed from source. |
| COMP-27 CaseSearchService | REST API only. No Kafka dependency. Confirmed from source. |
| COMP-28 DisplayCodeService | REST API only. No Kafka dependency. Confirmed from source. |
| COMP-29 FaxQueueService | REST API only. No Kafka dependency. Confirmed from source. |
| COMP-30 UserQueueSkillService | REST API only. No Kafka dependency. Confirmed from source. |
| COMP-31 BusinessRulesService | No Kafka dependency in pom.xml. Confirmed from source. |
| COMP-32 RulesService | No Kafka dependency in pom.xml. Confirmed from source. |
| COMP-34 MerchantTransactionService | REST API read-only. No Kafka dependency. Confirmed from source. |
| COMP-35 EncryptionService | REST API. No Kafka dependency. Confirmed from source. |
| COMP-36 TokenService | REST API. No Kafka dependency. Confirmed from source. |
| COMP-37 DocumentManagementService | REST API. No Kafka dependency confirmed from source. |
| COMP-38 APILogService | REST API sink. No Kafka dependency. Confirmed from source. |

---

*Last updated: April 2026*
*Version 2.0 — fully populated from all 40 component DRAFT files*
