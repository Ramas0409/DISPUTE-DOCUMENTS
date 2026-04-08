# WDP-KAFKA.md
**Worldpay Dispute Platform — Kafka Topic Registry**
*Version: 1.0 SKELETON | April 2026*
*Source: WDP-COMPONENTS.md Part 3 + component file enrichment (in progress)*

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

This document is populated in two passes:
- **Pass 1 (this skeleton):** Topic names, publishers, and consumers
  extracted from WDP-COMPONENTS.md Part 3 and cross-referenced with
  component descriptions.
- **Pass 2 (ongoing):** Full payload schemas, consumer group IDs,
  partition counts, and retention settings populated as each
  WDP-COMP-[NN]-*.md file is completed and confirmed.

Individual component files (WDP-COMP-[NN]-*.md) contain the Kafka
contracts specific to each component. This file is the cross-cutting
view — the complete topic map for the platform.

---

## How to Update This Document

When a WDP-COMP-[NN]-*.md file is completed:
1. Find the relevant topic rows in the Topic Registry below
2. Fill in any blank cells from the component's Kafka contract section
3. Cross-check the `Consumed by` column against the consumer's
   Block B section
4. Update the Consumer Group Registry with the confirmed group ID
5. Mark the topic row as ✅ CONFIRMED once both publisher and consumer
   component files are complete

---

## 1. MSK Infrastructure

**Platform:** AWS Managed Streaming for Apache Kafka (MSK)
**Architectural decision:** DEC-013 — Kafka (AWS MSK) as Event Streaming Platform

| Parameter | Value | Source |
|-----------|-------|--------|
| Cluster location | AWS (same region as EKS cluster) | Confirmed |
| Broker count | ⚠️ To be confirmed from infrastructure docs | PENDING |
| Availability zones | ⚠️ To be confirmed | PENDING |
| Authentication | AWS IAM (MSK IAM auth) | Confirmed from component analysis |
| Default retention | ⚠️ To be confirmed per topic | PENDING |
| Storage scaling | One-directional — cannot reduce after scaling up | Confirmed |
| Partition strategy | Merchant-scoped (DEC-003) | Confirmed |
| Offset commit standard | Manual commit after full processing (DEC-005) | Confirmed — deviations flagged per component |

⚠️ Broker count, AZ configuration, storage configuration, and per-topic
retention settings are to be confirmed from infrastructure documentation.
These were marked DRAFT in WDP-COMPONENTS.md Part 3.1.

---

## 2. Platform-Wide Kafka Standards

These decisions apply to every Kafka interaction in WDP unless a
component explicitly deviates (deviations are flagged in the relevant
WDP-COMP-[NN]-*.md file under Key Architectural Decisions).

| Standard | Decision | Reference |
|----------|----------|-----------|
| Partition key | merchantId on all topics — merchant-scoped isolation | DEC-003 |
| Offset commit | Manual commit after full processing completes | DEC-005 |
| Consumer failure | Database-backed error table (not Kafka DLQ topics) | Platform pattern |
| Idempotency key | UUID carried as Kafka message header `idempotency-key` | Platform pattern |
| Message ordering | Guaranteed within partition (per merchantId) | DEC-003 |
| Producer idempotence | enable.idempotence=true on producers where configured | Per component |

**Known deviations from DEC-005 (manual offset commit):**
- WDP-COMP-05 NAPDisputeEventProcessor — pre-ACK before processing
  (at-most-once). Deliberate decision to avoid redelivery.
- WDP-COMP-12 InboundDisputeEventScheduler — mark-before-send pattern
  (at-most-once). Database PUBLISHED status written before Kafka send.

**Known deviations from DEC-001 (transactional outbox):**
- WDP-COMP-04 NAPDisputeEventService — no outbox table. Publishes
  directly to Kafka on the HTTP thread. Events lost between enrichment
  completion and Kafka acknowledge are unrecoverable. No at-least-once
  delivery guarantee.

**Known deviations from DEC-003 (merchantId partition key):**
- WDP-COMP-04 NAPDisputeEventService — POST /event (new disputes) uses
  cardAcceptorCodeId as partition key, not merchantId. All other event
  types use merchantId. Partial deviation — whether deliberate or
  accidental is undocumented.

---

## 3. Topic Registry

### How to read this table

- **Publisher:** The component that writes to this topic.
  Reference format: COMP-NN (ComponentName)
- **Consumer(s):** The component(s) that read from this topic.
  Multiple consumers = multiple consumer groups on the same topic.
- **Partition key:** The Kafka message key used for partition routing.
- **Payload:** Link to the component file that owns the payload schema.
- **Confirmed:** ✅ when both publisher and consumer component files
  are complete. ⚠️ when source is WDP-COMPONENTS.md only (not yet
  verified from code).

---

| Topic Name | Publisher | Consumer(s) | Partition Key | Status | Payload Schema |
|------------|-----------|-------------|---------------|--------|----------------|
| `nap-dispute-events` | COMP-04 NAPDisputeEventService | COMP-05 NAPDisputeEventProcessor | merchantId (all events) / cardAcceptorCodeId (POST /event new disputes only) — see note [2] | ✅ Production | See WDP-COMP-04-NAP-DISPUTE-EVENT-SERVICE.md Block C (DRAFT) |
| `new-case-events` | COMP-12 InboundDisputeEventScheduler | COMP-14 CaseCreationConsumer | Compound key: networkCaseId + cardNetwork + platform (string concat from c_ntwk_case_id, c_case_ntwk, c_acq_platform) — ⚠️ DEC-003 deviation, not merchantId | ✅ Production | See WDP-COMP-12-INBOUND-EVENT-SCHEDULER.md Block C + WDP-COMP-14-CASE-CREATION-CONSUMER.md Block B |
| `case-evidence-events` | COMP-12 InboundDisputeEventScheduler | COMP-15 EvidenceConsumer | caseNumber (column `i_case`) if non-blank; else networkCaseId — ⚠️ DEC-003 deviation, not merchantId | ✅ Production | See WDP-COMP-15-EVIDENCE-CONSUMER.md Block B |
| `business-rules` | COMP-15 EvidenceConsumer (confirmed — WDP path, isMultiDocPending=false only); COMP-12 InboundDisputeEventScheduler (confirmed — Scheduler4, bre_orchestration_outbox rows with component=BUSINESS_RULES) ⚠️ COMP-14 CaseCreationConsumer candidacy still unverified — see note [1] | COMP-16 BusinessRulesProcessor | merchantId (inbound key from COMP-12/15 publishers) — consumer reads as `keyMerchantId` for logging only | ✅ Production | See WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md Block B |
| `outgoing-events` | COMP-16 BusinessRulesProcessor | COMP-18 NotificationOrchestrator | caseNumber — ⚠️ DEC-003 deviation, not merchantId. Published in finally block — always emitted if caseDetails + currentActionDetails non-null. ⚠️ DEC-001 deviation — no transactional outbox, direct synchronous publish. | ✅ Production | See WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md Block C |
| `internal-integration-events` | COMP-19 AcceptService, COMP-20 ContestService, COMP-16 BusinessRulesProcessor (conditional — UK/NAP path only, when action outcome requires network notification) | COMP-39 NAPOutcomeProcessor, COMP-40 VisaResponseQuestionnaire | caseNumber (COMP-16 confirmed) — ⚠️ DEC-003 deviation for COMP-16 publishes. COMP-19/20 partition key ⚠️ TBC | ✅ Production | See WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md Block C (COMP-16 portion) |
| `case-action-events` (expiry) | COMP-18 NotificationOrchestrator (confirmed — EXPIRY_EVENT rows, Filter 1 routing) | COMP-17 CaseExpiryUpdateConsumer | Pass-through of `KafkaHeaders.RECEIVED_KEY` from upstream — ⚠️ DEC-003 likely deviation | ✅ Production | See WDP-COMP-17-CASE-EXPIRY-CONSUMER.md Block B + WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md Block C |
| `external-request-events` | COMP-18 NotificationOrchestrator | COMP-41 ThirdPartyNotificationConsumer, COMP-42 BENConsumer, COMP-44 EDIAConsumer | merchantId | ✅ Production | ⚠️ See WDP-COMP-18 (PENDING) |
| `core-request-events` | COMP-18 NotificationOrchestrator | COMP-43 CoreNotificationConsumer | merchantId | ✅ Production | ⚠️ See WDP-COMP-18 (PENDING) |
| `EDIA-events` | COMP-44 EDIAConsumer | EDIA Platform (external) | TBD | 🔴 Planned | ⚠️ PENDING — EDIA format TBD |

**Notes:**

[1] **business-rules topic publisher — partially resolved:** COMP-15
EvidenceConsumer is confirmed as a publisher (WDP path only, when
isMultiDocPending=false). Publishes synchronously inside @Transactional —
DEC-001 deviation. COMP-14 CaseCreationConsumer candidacy still to be
verified via Copilot CLI follow-up: "Does CaseCreationConsumer publish to
the business-rules Kafka topic after case creation? If yes, what is the
config key and message key?" This must be fully resolved before
WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md can be marked COMPLETE.
[2] **nap-dispute-events partition key deviation from DEC-003:**
The partition key for new dispute events (POST /event) is
cardAcceptorCodeId, not merchantId. All other event types (case
update, win/loss outcome) correctly use merchantId. This is a
confirmed partial deviation from DEC-003. Whether deliberate or
accidental is undocumented. See WDP-COMP-04 Key Architectural
Decisions. ⚠️ Decommission-scoped — this topic will be retired
when the NAP/WPG inbound path is decommissioned.

---

## 4. Consumer Group Registry

Consumer group IDs are injected via Kubernetes secrets in most components
and are not committed to source repositories. This table captures group IDs
as they are confirmed from component analysis or environment inspection.

| Consumer Group ID | Topic | Component | Offset Strategy | Confirmed |
|-------------------|-------|-----------|-----------------|-----------|
| `${kafka_group_id}` — PENDING confirmation | `nap-dispute-events` | COMP-05 NAPDisputeEventProcessor | Pre-ACK (deviation from DEC-005) | ⚠️ Injected via K8s secret |
| `new-case-events-group` (prod) / `new-case-events-group-dev` (dev) | `new-case-events` | COMP-14 CaseCreationConsumer | Pre-ACK — MANUAL_IMMEDIATE before processing (deviation from DEC-005) | ✅ Confirmed from COMP-14 source |
| `case-evidence-events-group` | `case-evidence-events` | COMP-15 EvidenceConsumer | Pre-ACK — MANUAL_IMMEDIATE, syncCommits=true, offset committed before processing (deviation from DEC-005) | ✅ Confirmed from COMP-15 source |
| `business-rules-processor-group` | `business-rules` | COMP-16 BusinessRulesProcessor | Pre-ACK — MANUAL_IMMEDIATE, offset committed before all processing (deviation from DEC-005). Concurrency: 1 (default). | ✅ Confirmed from COMP-16 source |
| `outgoing-events-group` | `outgoing-events` | COMP-18 NotificationOrchestrator | MANUAL_IMMEDIATE, syncCommits=true — ⚠️ ACK committed after outbox INSERT but BEFORE Kafka publishes (mid-flow). DEC-005 deviation. Concurrency: 1 (default). | ✅ Confirmed from COMP-18 source |
| ⚠️ PENDING | `internal-integration-events` | COMP-39 NAPOutcomeProcessor | ⚠️ TBC | ⚠️ PENDING |
| ⚠️ PENDING | `internal-integration-events` | COMP-40 VisaResponseQuestionnaire | ⚠️ TBC | ⚠️ PENDING |
| `case-action-events-group` | `case-action-events` | COMP-17 CaseExpiryUpdateConsumer | MANUAL_IMMEDIATE, syncCommits=true — ⚠️ ACK timing inconsistent between Path A (pre-ACK before writes) and Path B (mid-flow ACK after outbox INSERT, before case_expiry write). Deviation from DEC-005. | ✅ Confirmed from COMP-17 source |
| ⚠️ PENDING | `external-request-events` | COMP-41 ThirdPartyNotificationConsumer | ⚠️ TBC | ⚠️ PENDING |
| ⚠️ PENDING | `external-request-events` | COMP-42 BENConsumer | ⚠️ TBC | ⚠️ PENDING |
| ⚠️ PENDING | `external-request-events` | COMP-44 EDIAConsumer | ⚠️ TBC | 🔴 Planned |
| ⚠️ PENDING | `core-request-events` | COMP-43 CoreNotificationConsumer | ⚠️ TBC | ⚠️ PENDING |

*Consumer group IDs will be confirmed during Copilot CLI component analysis
and populated as each WDP-COMP-[NN]-*.md file is completed.*

---

## 5. Outbox Tables That Feed Kafka

WDP uses the transactional outbox pattern (DEC-001) extensively. These
tables are written by inbound processors and batch jobs, then read by
InboundDisputeEventScheduler (COMP-12) to publish to Kafka. They are
not Kafka topics, but they are functionally part of the Kafka flow.

| Outbox Table | Written by | Read / Published by | Kafka Topic Target |
|--------------|------------|---------------------|--------------------|
| `wdp.chbk_outbox_row` | COMP-07 VisaDisputeBatch, COMP-08 FirstChargebackBatch, COMP-09 CaseFillingBatch, COMP-11 FileProcessor | COMP-12 InboundDisputeEventScheduler | `new-case-events`, `case-evidence-events` |
| `wdp.outgoing_event_outbox` | ⚠️ Publisher TBC — Scheduler3 in COMP-12 reads this | COMP-12 InboundDisputeEventScheduler (Scheduler3) | ⚠️ Topic TBC |
| `wdp.bre_orchestration_outbox` | ⚠️ Publisher TBC — Scheduler4 in COMP-12 reads this | COMP-12 InboundDisputeEventScheduler (Scheduler4) | ⚠️ Topic TBC |
| `wdp.file_evidence` | COMP-11 FileProcessor | COMP-12 InboundDisputeEventScheduler (Scheduler5 — error report only) | N/A — evidence tracking only |
| `wdp.notification_orchestration_outbox` | ⚠️ TBC | COMP-12 InboundDisputeEventScheduler (Scheduler4) | `outgoing-events` |

⚠️ **Open question:** Schedulers 3 and 4 in COMP-12 query
`wdp.outgoing_event_outbox` and `wdp.bre_orchestration_outbox` for
PROCESSING status only (not PENDING). If no other component writes
PROCESSING rows to these tables, these schedulers may be processing
a gap. This requires architect confirmation before any changes to
these tables. See COMP-12 Risks & Constraints.

---

**Components confirmed Kafka-free (no producer, consumer, or dependency):**

| Component | Confirmation basis |
|-----------|-------------------|
| COMP-01 API Gateway | No Kafka dependency in pom.xml. No KafkaTemplate or @KafkaListener. Confirmed from source. |
| COMP-02 UAMS | No Kafka dependency in pom.xml. No KafkaTemplate or @KafkaListener. Confirmed from source. |
| COMP-03 CHAS | No spring-kafka dependency in pom.xml. No KafkaTemplate or @KafkaListener anywhere in codebase. Confirmed from source. |
| COMP-08 FirstChargebackBatch | No spring-kafka dependency in pom.xml. No KafkaTemplate or @KafkaListener. Kafka dependencies staged (spring-kafka, aws-msk-iam-auth) but zero producers or consumers wired. Confirmed Kafka-free from source. |
| COMP-09 CaseFillingBatch | No spring-kafka dependency confirmed. No KafkaTemplate or @KafkaListener. Kafka dependencies may be staged in pom.xml (consistent with COMP-07/08 pattern) but zero producers or consumers wired. Confirmed Kafka-free from source. |
| COMP-11 FileProcessor | No Kafka dependency in pom.xml. No KafkaTemplate or @KafkaListener. SQS-triggered — uses Spring Cloud AWS SQS listener, not Kafka. Zero Kafka infrastructure anywhere in codebase. Confirmed Kafka-free from source. |

*This document is a living registry. Populate it as each component
file is completed. Mark topic rows ✅ CONFIRMED once both publisher
and consumer component files are architect-confirmed.*

*Last updated: April 2026*
*Enrichment in progress — populating as WDP-COMP-[NN]-*.md files are completed*
