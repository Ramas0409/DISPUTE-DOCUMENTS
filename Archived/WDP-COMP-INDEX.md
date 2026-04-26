# WDP-COMP-INDEX.md
**Worldpay Dispute Platform — Master Component Registry**
*Version: 2.0 | April 2026*
*Updated: All component files uploaded and verified April 2026*

---

## Purpose of This Document

This is the navigation hub for all WDP component documentation.
It maps every component to its file, records production and
documentation status at a glance, and provides a one-line
responsibility summary for each component.

Claude uses this file to navigate the component knowledge base.
You use it to track migration progress from WDP-COMPONENTS.md
to individual component files, and to assign numbers to new
components added each quarter.

This document does NOT contain component detail. All detail
lives in the individual WDP-COMP-[NN]-[NAME].md files.

---

## Numbering Rules

- Numbers are permanent. Once assigned, a number never changes
  and is never reused, even if the component is retired.
- Numbers are sequential in the order components were first
  documented. They do not imply architectural hierarchy.
- New components added each quarter get the next available number.
- The number is the stable identifier. The name segment of the
  filename is human-readable context and may be updated if a
  component is renamed.
- Format: two-digit zero-padded integer — 01, 02, ... 09, 10, 11...
  Extend to three digits (001...) only if the platform exceeds 99
  components.

---

## What Is and Is Not a Component

**Gets a component number and file:**
Any independently deployable unit — a microservice, a batch job,
a Kubernetes Deployment or CronJob, a UI application.

**Does NOT get a component number:**
- The Kafka Event Bus (Part 3 in WDP-COMPONENTS.md) — this is
  infrastructure configuration and a topic registry. Documented
  in WDP-KAFKA.md.
- Acquiring Platform Integration contexts (Part 6 in
  WDP-COMPONENTS.md) — these are integration patterns, not
  deployable components. Documented in WDP-INTEGRATIONS.md.
- UI sections within a portal (Disputes Section, Queues Section,
  etc.) — these are functional areas within a portal application,
  not separate deployables. Documented as sections within the
  relevant portal component file (WDP-COMP-49 or WDP-COMP-50).

**Special note on FileAcknowledgementProcessor:**
Sections 2.3.4 and 5.3.2 in WDP-COMPONENTS.md both describe
ACK file generation behaviour that maps to the same deployable:
`wdp-evidence-ack-scheduler`. These have been consolidated into
a single component (13). Section 5.3.2 in WDP-COMPONENTS.md
is a cross-reference, not a separate service.

---

## Status Key

**Production status:**
- ✅ Production — live in production
- 🔄 In Progress — being built or partially live
- 🔴 Planned — not yet started

**Documentation status:**
- ✅ COMPLETE — individual component file exists, architect-confirmed
- 📝 DRAFT — individual component file created from Copilot CLI analysis, architect confirmation pending
- 📋 PENDING — not yet migrated to individual file
- ⬜ NOT STARTED — no content exists yet (no repo or planned feature)
- 🔲 UI — tracked as separate action item

---

## Component Registry

### Access & API Layer

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 01 | API Gateway | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-01-API-GATEWAY.md |
| 02 | UserAccessManagementService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-02-UAMS.md |
| 03 | CoreHierarchyAuthorizationService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-03-CHAS.md |

**01 — API Gateway**
Single entry point for all WDP traffic. Runs a 4-layer JWT authentication,
request logging, region-role authorization, and case-level entity authorization
filter pipeline before proxying requests to backend microservices. URL-based
routing configuration loaded from database at startup.

**02 — UserAccessManagementService (UAMS)**
Access control and user lifecycle management for the NAP platform. Owns the
merchant entity hierarchy and ACL. Exposes the /authorize endpoint called by
the API Gateway for NAP platform case-level authorization. Proxies all user
lifecycle operations to SunGard IdP.

**03 — CoreHierarchyAuthorizationService (CHAS)**
Read-only authorization service for PIN, CORE, VAP, and LATAM platforms.
Validates JWT entity claims against Core enterprise merchant hierarchy in
IBM DB2. Exposes the /authorize endpoint called by the API Gateway for all
non-NAP platform case-level authorization. Owns no data and performs no writes.

---

### Inbound Processing — NAP/WPG Path

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 04 | NAPDisputeEventService ⚠️ Decommission-scoped | REST API + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-04-NAP-DISPUTE-EVENT-SERVICE.md |
| 05 | NAPDisputeEventProcessor ⚠️ Decommission-scoped | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-05-NAP-DISPUTE-EVENT-PROCESSOR.md |
| 06 | NAPDisputeDeclineBatch ⚠️ Decommission-scoped | Batch/Scheduler | ✅ Production | 📝 DRAFT | WDP-COMP-06-NAP-DISPUTE-DECLINE-BATCH.md |

**04 — NAPDisputeEventService**
REST-to-Kafka bridge for inbound NAP dispute events. Receives SRV-116 events
and operator responses via REST, enriches via synchronous lookups against
internal WDP services, and publishes enriched NapEvent to nap-dispute-events
topic. Document upload path proxies directly to DocumentManagementService
without Kafka involvement. Stateless — no database.

**05 — NAPDisputeEventProcessor**
Kafka consumer on nap-dispute-events topic. Translates NAP dispute notifications
into case management actions via REST calls to downstream services. Classifies
events as SRV116, WIN/LOSS, or SRV118 by field inspection. Uses at-most-once
offset commit (DEC-005 deviation). Database-backed DLQ — no Kafka dead-letter
topic. Currently in hybrid CB911 migration state.

**06 — NAPDisputeDeclineBatch**
Spring Batch job that polls nap.action table for open PAB actions, queries
Visa network via Visa Adapter HyperSearch, and creates IDCL draft actions for
pre-arbitration declined cases. VISA and NAP platform only. Runs as Kubernetes
Deployment with internal Spring @Scheduled cron. No Kafka involvement.

---

### Inbound Processing — Card Network Batch Path

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 07 | VisaDisputeBatch | Batch/Scheduler | ✅ Production | 📝 DRAFT | WDP-COMP-07-VISA-DISPUTE-BATCH.md |
| 08 | FirstChargebackBatch | Batch/Scheduler | ✅ Production | 📝 DRAFT | WDP-COMP-08-FIRST-CHARGEBACK-BATCH.md |
| 09 | CaseFillingBatch | Batch/Scheduler | ✅ Production | 📝 DRAFT | WDP-COMP-09-CASE-FILLING-BATCH.md |

**07 — VisaDisputeBatch**
Origin of all Visa dispute data in WDP. Polls seven Visa RTSI queue types
sequentially via DataPower Gateway, enriches each event via HyperSearch and
internal WDP services, encrypts PAN, and writes PENDING rows to the
chbk_outbox_row transactional outbox. Does not publish to Kafka directly.
Sequential single-threaded processing — all 7 queues in one job execution.

**08 — FirstChargebackBatch**
Origin of all MasterCard first chargeback data in WDP. Polls MCM unworked
first chargeback queue via DataPower, fetches claim detail and settled
transaction PAN, encrypts PAN, and writes PENDING rows to chbk_outbox_row.
Acknowledges processed items back to MCM. Runs every 5 minutes. Replica
count confirmed as 1 — replicas > 1 creates parallel polling risk.

**09 — CaseFillingBatch**
Handles MasterCard dispute lifecycle stages after first chargeback: PAB,
ARB, PRA, AII, AIM. Polls MCM ReceiverCaseFiling queue via DataPower,
determines dispute stage from claim detail fields, encrypts PAN for new
claims, and writes to chbk_outbox_row. Stage determination uses priority-
ordered branch logic on caseType + rulingDate + issuerAction.

---

### Inbound Processing — File-Based Path

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 10 | DM Mainframe | External Infrastructure | ✅ Production | 📋 PENDING | WDP-COMP-10-DM-MAINFRAME.md |
| 11 | FileProcessor | Batch/Scheduler | ✅ Production | 📝 DRAFT | WDP-COMP-11-FILE-PROCESSOR.md |
| 12 | InboundDisputeEventScheduler | Batch/Scheduler + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-12-INBOUND-EVENT-SCHEDULER.md |
| 13 | FileAcknowledgementProcessor | Batch/Scheduler | ✅ Production | ✅ COMPLETE | WDP-COMP-13-FILE-ACK-PROCESSOR.md |

**10 — DM Mainframe**
On-premise enterprise mainframe that bridges file-based card network and
merchant integrations with WDP. Not owned by the WDP team — shared enterprise
infrastructure. Receives inbound files from Amex, Discover, PIN Networks,
CapitalOne, Walmart, and Meijer via Sterling Mailbox. Routes outbound response
files back to those partners. All transfers via mainframe-to-mainframe protocol
through Sterling Mailbox. WDP dependency on enterprise SLA.

**11 — FileProcessor**
Primary inbound file-ingestion component. Triggered by SQS events when ZIP
files land in S3. Identifies source from filename, parses six source types
(DWSG, DBLK, DISR, MFAD, DCPO, DNWK), writes to three PostgreSQL outbox
tables (file_job, chbk_outbox_row, file_evidence), stages evidence documents
to S3, and encrypts PAN for DNWK sources. Does not publish to Kafka — the
outbox is the handoff to InboundDisputeEventScheduler.

**12 — InboundDisputeEventScheduler**
Transactional outbox publisher. Runs five schedulers that read PENDING rows
from four outbox tables and publish to the appropriate Kafka topics. Serves
as the Kafka publishing layer for VisaDisputeBatch, FirstChargebackBatch,
CaseFillingBatch, and FileProcessor — none of those write to Kafka directly.
Uses mark-before-send (at-most-once) delivery pattern. Replica count > 1
creates race condition risk — no distributed locking.

**13 — FileAcknowledgementProcessor**
Generates outbound acknowledgement files for inbound file jobs that required
confirmation. Polls file_job table for completed jobs with ack_required=true,
reads per-row detail from chbk_outbox_row, formats ACK files in four merchant-
specific formats (Walmart SigCap, Meijer, CapitalOne CMRTR, CapitalOne BJWC),
and uploads to S3. Documented in WDP-COMPONENTS.md sections 2.3.4 and 5.3.2
— these are the same deployable (wdp-evidence-ack-scheduler).

---

### Core Processing — Event Consumers

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 14 | CaseCreationConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-14-CASE-CREATION-CONSUMER.md |
| 15 | EvidenceConsumer | Kafka Consumer + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-15-EVIDENCE-CONSUMER.md |
| 16 | BusinessRulesProcessor | Kafka Consumer + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md |
| 17 | CaseExpiryUpdateConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-17-CASE-EXPIRY-CONSUMER.md |
| 18 | NotificationOrchestrator | Kafka Consumer + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md |

**14 — CaseCreationConsumer**
Primary case creation component for all non-NAP dispute events. Consumes
from new-case-events topic. Enriches events with merchant and transaction
data from the relevant acquiring platform API (CORE via MerchantTransactionService,
LATAM and VAP via direct API calls, PIN path TBC). Creates dispute cases in
WDP Core via CaseManagementService. Handles transient PAN decryption for
enrichment calls — clear PAN never persisted. Pre-ACK (DEC-005 deviation).

**15 — EvidenceConsumer**
Consumes from case-evidence-events topic and attaches evidence documents to
existing dispute cases. Two paths: WDP path (PIN always; CORE when
coreMigrationFlag=true) uploads via DocumentManagementService and publishes
to business-rules; V3 legacy path (CORE when flag=false) queries IBM DB2 and
uploads to V3 Core endpoint. Pre-ACK (DEC-005 deviation). ⚠️ MISCDOC/DRFTDOC
document types silently unprocessed — known gap.

**16 — BusinessRulesProcessor**
Kafka consumer that executes configured business rules against dispute events.
The execution engine — distinct from BusinessRulesService which manages rule
definitions. Reads rules directly from Aurora PostgreSQL (no API call to
BusinessRulesService). Publishes processed events to outgoing-events and
conditionally to internal-integration-events (UK/NAP path). Pre-ACK
(DEC-005 deviation). ⚠️ DEC-011 BRE step checkpointing NOT implemented —
aspirational design only.

**17 — CaseExpiryUpdateConsumer**
Consumes from case-action-events (expiry) topic and maintains wdp.case_expiry
table — the active expiry schedule per case and action. State-machine driven
via internal apiName token. Uses wdp.outgoing_event_outbox for idempotency and
predecessor control. Terminal consumer — no Kafka publish. ⚠️ ACK timing
inconsistent between Path A (pre-ACK) and Path B (mid-flow ACK).

**18 — NotificationOrchestrator** *(repository: wp-mfd/wdp-outgoing-consumer)*
Central outbound routing component. Consumes from outgoing-events topic and
fans out to up to four simultaneous outputs via four independent filter methods.
Publishes to case-action-events, core-request-events, and external-request-events.
Also writes to wdp.file_generation_event to stage requests for file-based output
components. Uses wdp.bre_orchestration_outbox for idempotency and retry (component=
NOTIFICATION_ORCHESTRATOR rows). ACK committed after outbox INSERT but before
Kafka publishes (DEC-005 deviation). Does NOT publish to internal-integration-events.

---

### Core Processing — Case Action Services

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 19 | AcceptService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-19-ACCEPT-SERVICE.md |
| 20 | ContestService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-20-CONTEST-SERVICE.md |
| 21 | ChargebackService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-21-CHARGEBACK-SERVICE.md |
| 22 | DisputeService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-22-DISPUTE-SERVICE.md |
| 23 | CaseManagementService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-23-CASE-MANAGEMENT-SERVICE.md |
| 24 | CaseActionService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-24-CASE-ACTION-SERVICE.md |
| 25 | NotesService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-25-NOTES-SERVICE.md |
| 26 | QuestionnaireService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-26-QUESTIONNAIRE-SERVICE.md |

**19 — AcceptService**
Processes merchant decisions to accept a dispute. Calls Visa and MasterCard
APIs directly and synchronously. Publishes AcceptEvent to internal-integration-events
(caseNumber key — DEC-003 deviation). For NAP disputes, NAPOutcomeProcessor
delivers the outcome to NAP-DPS. No Kafka DLQ or outbox — DEC-001 deviation.

**20 — ContestService**
Processes merchant decisions to contest a dispute. Calls Visa and MasterCard
APIs directly and synchronously. Publishes ContestEvent to internal-integration-events
(merchantId key — DEC-003 compliant). Up to 3 publishes per request when CRMR
action code present. No Kafka DLQ or outbox — DEC-001 deviation.

**21 — ChargebackService**
Sole externally-exposed WDP REST API accessible to merchants via APIGEE and
Akamai. Handles both dispute read operations (case search, detail, documents)
and action operations (contest, accept, add note, change owner, document upload).
Platform routing derived entirely from inbound caseId format. Two runtime
authorization modes toggled by entitlement_flag. Third-party systems
(SignifyD, JustAI, BEN) call back via this service after receiving notifications.

**22 — DisputeService**
Read-and-orchestration layer only — owns no database state, performs no writes.
Two independent endpoints: POST /summary (dispute summary aggregation from WDP
Aurora or IBM DB2 depending on core_migration_status flag) and POST
/{platform}/cases/{caseNumber}/documents (internal-firm document upload
orchestration). Kafka producer to business-rules is wired but call site is
commented out in production.

**23 — CaseManagementService**
Authoritative owner of the dispute case record across all five platforms (NAP,
PIN, CORE, VAP, LATAM). Single write target for case creation and updates.
Owns separate nap and wdp PostgreSQL schemas — never cross-schema in one
transaction. Publishes BusinessRuleEvent to business-rules Kafka topic
(caseNumber key — DEC-003 deviation) after every material write.
⚠️ DEC-004 violation: clear PAN written to persistent storage on standard
case creation — encryption only occurs during transaction enrichment flow.

**24 — CaseActionService**
Handles all case action lifecycle operations across NAP and US platforms —
creating, updating, and closing actions; ownership transfers; case open/close
transitions. Writes wdp.case, wdp.action, nap.case, nap.action, and
conditionally wdp.chbk_outbox_row and wdp.notes in the same transaction.
Publishes to business-rules topic and a second ActionEvent topic (${kafka.topic}
— consumer TBC). ⚠️ No RBAC enforcement — any authenticated caller can modify
any case action.

**25 — NotesService**
Platform-aware notes persistence service for all WDP acquiring platforms. Dual
schema (nap/wdp) — separate JPA entity managers, transaction managers, and
datasources, never mixed. Publishes AddNotesBREvent to business-rules Kafka
topic for all non-SNOTE note types. ⚠️ DEC-001 deviation — DB write and Kafka
publish share @Transactional boundary but are not atomically coupled.

**26 — QuestionnaireService**
Persistence and retrieval store for all dispute-response questionnaires. Two
controller families: VisaQuestionnaireController (Representment, Pre-Compliance,
Pre-Arbitration, Allocation stages) and DisputeQuestionnaireController (non-Visa
+ document-status lookup). Serialises full questionnaire payload as JSON blob
into wdp.disputes_questionnaire. No Kafka.
⚠️ POST idempotency gap — duplicate POSTs insert new rows.

---

### Core Processing — Search & Display

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 27 | CaseSearchService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-27-CASE-SEARCH-SERVICE.md |
| 28 | DisplayCodeService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-28-DISPLAY-CODE-SERVICE.md |

**27 — CaseSearchService**
Platform-wide read layer for dispute case data across all five acquiring
platforms. 14 active REST endpoints across six controllers. Platform routing
via {platform}/{region} path variable selecting between nap.* and wdp.* schemas.
Case detail endpoints use parallel async fan-out (Spring @Async, 10-thread pool)
calling up to five downstream services concurrently. Display code lookups
in-memory cached for JVM lifetime.

**28 — DisplayCodeService**
Reference-data lookup hub. Returns structured code/description lists for ~40
code domain types given requested domain names and optional platform filter.
Secondary function resolves UI tab permissions from wdp.dispute_static_tabs_rules.
All results cached in-memory (no TTL, no eviction — pod restart only).
⚠️ Correction: does NOT determine TIER1 eligibility — it returns raw code lists;
eligibility logic belongs to the calling service.

---

### Core Processing — Queue & Workflow Services

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 29 | FaxQueueService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-29-FAX-QUEUE-SERVICE.md |
| 30 | UserQueueSkillService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-30-USER-QUEUE-SKILL-SERVICE.md |

**29 — FaxQueueService**
Manages full lifecycle of inbound fax documents received from merchants. Backend
for the Fax Queue section of the WDP Ops Portal. Writes to legacy MS SQL Server
(dbo.IncomingFaxes) and audit records to WDP PostgreSQL (WDP.FAX_ACTION). Also
bundles an eViewer License Management proxy as a secondary controller group.
Provides three reporting endpoints including CSV email delivery.

**30 — UserQueueSkillService**
Manages users, queues, and skills for the WDP platform. Owns upsert of user
records from JWT claims on portal session start. Manages queue creation and
assignment, skill creation and assignment, and queue criterion rules.
Auto-enrols internal users with NAP queue roles into SKILLS_INTERNAL queues
on first insert.

---

### Core Processing — Rules & Configuration

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 31 | BusinessRulesService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-31-BUSINESS-RULES-SERVICE.md |
| 32 | RulesService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-32-RULES-SERVICE.md |

**31 — BusinessRulesService**
CRUD management service for business rules — owns nap.rules, nap.rule_criterion,
nap.rule_action and wdp equivalents. Portal UIs call this to create, update,
enable/disable, and delete rules. Read-only for br_case_audit_log
(GET /audit-log). No Kafka. Rule execution is performed by BusinessRulesProcessor
(COMP-16) reading directly from these tables — BusinessRulesService is never
called at execution time.

**32 — RulesService**
Read-only configuration service. Serves workflow rule configuration, action
configuration, and stage-locking rules used by the dispute portals. Owns no
writes — all tables are populated via database migrations or admin tooling.
Used by portal UIs to determine available actions and locked-out case stages
for non-migrated merchants.

---

### Core Processing — Organisation & User Services

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 33 | OrgManagementService | REST API | ✅ Production | ⬜ NOT STARTED | WDP-COMP-33-ORG-MANAGEMENT-SERVICE.md |
| 34 | MerchantTransactionService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-34-MERCHANT-TRANSACTION-SERVICE.md |

**33 — OrgManagementService**
Manages organisational hierarchies within WDP. Handles merchant account
configuration, org-level settings, and routing rule configuration at the
organisation level. ⚠️ GitHub repository not found — documentation deferred.

**34 — MerchantTransactionService**
Provides merchant and transaction data enrichment for CORE acquiring platform
disputes. Supports six endpoint families: external transaction lookup
(Visa/CapOne/international), card activity summary, authorisation details,
chargeback details, settlement details, and case search. Read-only across all
datasources — owns no database state. Calls EncryptionService (COMP-35) to
decrypt HPAN before enrichment API calls.
⚠️ DEC-014 deviation: no timeouts, no circuit breakers on any of 10 outbound
dependencies.

---

### Core Processing — Supporting Services

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 35 | EncryptionService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-35-ENCRYPTION-SERVICE.md |
| 36 | TokenService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-36-TOKEN-SERVICE.md |
| 37 | DocumentManagementService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-37-DOCUMENT-MANAGEMENT-SERVICE.md |
| 38 | APILogService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-38-API-LOG-SERVICE.md |

**35 — EncryptionService**
Sole component authorised to handle plaintext PAN data in WDP. Implements
two-token PAN strategy (DEC-007) — HPAN for deterministic lookups, EPAN for
reversible storage. Uses AWS KMS with 6-hour DEK cache (DEC-008). Global
dependency — all inbound processing paths that touch PAN halt if this service
is unavailable.

**36 — TokenService**
Centralised JWT token management for all WDP consumers and batch jobs. Two-layer
cache: AWS ElastiCache Redis (primary) and Spring in-memory OAuth2 store
(secondary). Falls through to IDP client_credentials grant only on full cache
miss. ⚠️ Redis hash must be written by an external component not in this
repository — identity of that writer is an open question. Note: no relation to
PAN tokenisation — JWT management only.

**37 — DocumentManagementService**
Handles evidence document storage and retrieval. Documents stored in S3 with
metadata in DynamoDB. Called by EvidenceConsumer (inbound file evidence),
VisaResponseQuestionnaire (Visa contest questionnaires), NAPDisputeEventService
(NAP document attachments), and merchant UI submissions. Only WDP component
with a DynamoDB dependency.

**38 — APILogService**
Centralised error-log sink. Callers POST structured error records to the single
endpoint POST /log on error conditions. No AOP inside this service — callers
hold the catch-block logic and invoke this service directly via REST. Four
confirmed callers: AcceptService (COMP-19), ContestService (COMP-20),
CaseActionService (COMP-24), DocumentManagementService (COMP-37).
⚠️ Correction: prior description of AOP and LoggingException trigger is
incorrect — confirmed from source.

---

### Outbound Processing — Card Network Response

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 39 | NAPOutcomeProcessor | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-39-NAP-OUTCOME-PROCESSOR.md |
| 40 | VisaResponseQuestionnaire | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-40-VISA-RESPONSE-QUESTIONNAIRE.md |

**39 — NAPOutcomeProcessor**
Consumes internal-integration-events and delivers dispute outcomes to NAP-DPS
for NAP platform money movement. Filters to platform=NAP only — all other
platform events silently discarded. Processes both SRV116 (accept/contest)
and SRV117/SRV118 events. Built-in prior-error reprocessing on each message
cycle. ⚠️ Pre-ACK (DEC-005 deviation). ⚠️ notesLookup and checkCRMRAction
both commented out in production. Planned migration to EDIA route.

**40 — VisaResponseQuestionnaire**
Consumes internal-integration-events and retrieves the Visa questionnaire
submitted as part of a merchant contest. Filters to non-null visaResponseIds
only — AcceptService events silently discarded. Calls Visa RTSI API to retrieve
questionnaire, then stores via DocumentManagementService (S3 + DynamoDB).
⚠️ Pre-ACK (DEC-005 deviation).

---

### Outbound Processing — Notification Consumers

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 41 | ThirdPartyNotificationConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-41-THIRD-PARTY-NOTIFICATION-CONSUMER.md |
| 42 | BENConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-42-BEN-CONSUMER.md |
| 43 | CoreNotificationConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-43-CORE-NOTIFICATION-CONSUMER.md |
| 44 | EDIAConsumer | Kafka Consumer + Kafka Producer | 🔴 Planned | ⬜ NOT STARTED | WDP-COMP-44-EDIA-CONSUMER.md |

**41 — ThirdPartyNotificationConsumer**
Consumes external-request-events and delivers dispute event notifications to
Signifyd via REST API. Applies own filtering to determine which events require
third-party notification. After receiving notification Signifyd calls back
ChargebackService for dispute details. ⚠️ JustAI is planned — no JustAI
reference exists anywhere in the current codebase. Signifyd is the sole live
vendor. Confirmed from WDP-INTEGRATIONS.md v2.0.

**42 — BENConsumer**
Consumes external-request-events and delivers dispute lifecycle notifications
to BEN merchant notification platform via Kafka publish to a BEN-owned MSK
cluster (separate SASL/JAAS config from WDP MSK). ⚠️ There is no REST or
webhook call to BEN — delivery is Kafka-only. Merchants receive notifications
via BEN and call back ChargebackService for dispute details and actions.
Confirmed from WDP-INTEGRATIONS.md v2.0.

**43 — CoreNotificationConsumer**
Consumes core-request-events and writes dispute case and occurrence records
to the CORE platform IBM DB2 database (BC.TBC_DM_CASE, BC.TBC_DM_OCCUR,
BC.TBC_DM_NOTES). Full enrichment via REST calls to CaseManagementService,
CaseActionsService, NotesService, and EncryptionService before DB2 write.
Uses wdp.outgoing_event_outbox (channel_type=CORE_EVENTS) for idempotency
and retry. Sole WDP component that writes to IBM DB2 Core platform.

**44 — EDIAConsumer** *(Planned)*
WDP-owned anti-corruption layer between WDP internal event format and the
enterprise EDIA Kafka streaming platform. Will consume from external-request-events,
translate WDP internal dispute events to EDIA enterprise format, and publish
to the EDIA platform Kafka topic. Strategic outbound route for NAP, LATAM,
and VAP acquiring platforms.

---

### Outbound Processing — File Generation

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 45 | CapitalOneResponseFileProcessor | Batch/Scheduler | ✅ Production | ⬜ NOT STARTED | WDP-COMP-45-CAPONE-RESPONSE-FILE-PROCESSOR.md |
| 46 | NetworkResponseFileProcessor | Batch/Scheduler | ✅ Production | ⬜ NOT STARTED | WDP-COMP-46-NETWORK-RESPONSE-FILE-PROCESSOR.md |
| 47 | DialoguIssuerDocumentProcessor | Batch/Scheduler | ✅ Production | ⬜ NOT STARTED | WDP-COMP-47-DIALOGU-ISSUER-DOC-PROCESSOR.md |
| 48 | NYCEFileGenerationProcessor | Batch/Scheduler | 🔴 Planned | ⬜ NOT STARTED | WDP-COMP-48-NYCE-FILE-GENERATION-PROCESSOR.md |

**45 — CapitalOneResponseFileProcessor**
Reads from wdp.file_generation_event DB table and generates CapitalOne response
files. Places files in S3 /outbound/capitalOne/ for ControlM to transfer to
Sterling Mailbox for delivery to CapitalOne via DM Mainframe.
⬜ Not yet documented — planned next sprint.

**46 — NetworkResponseFileProcessor**
Reads from wdp.file_generation_event DB table and generates four network response
files: Amex, AmexHybrid, Discover, and DiscoverHybrid. Each placed in its own
dedicated S3 /outbound folder. DiscoverHybrid uses a special on-premise File
Transfer Batch pull via SFTP rather than the standard ControlM push flow.
⬜ Not yet documented — planned next sprint.

**47 — DialoguIssuerDocumentProcessor**
Reads from wdp.file_generation_event DB table and generates a ZIP file containing
all issuer documents received by WDP. Places ZIP in S3 /outbound/dialogu/ for
delivery to merchants via Sterling SFTP to Dialogue.
⬜ Not yet documented — planned next sprint.

**48 — NYCEFileGenerationProcessor** *(Planned)*
Will generate outbound files for PIN Networks (NYCE) and place them in
S3 /outbound/pinNetworks/ for ControlM to transfer to Sterling → DM Mainframe
→ PIN Networks.

---

### UI Layer

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 49 | WDP Merchant Portal | UI Application | ✅ Production | 🔲 UI — separate action | WDP-COMP-49-MERCHANT-PORTAL.md |
| 50 | WDP Ops Portal | UI Application | ✅ Production | 🔲 UI — separate action | WDP-COMP-50-OPS-PORTAL.md |

**49 — WDP Merchant Portal**
Merchant-facing UI. Allows merchants to view disputes, take actions, submit
evidence, and manage their organisation and users. Routes through Akamai for
CDN and edge security. Includes Disputes, User Management, and Org Management
sections. Dashboard section planned.

**50 — WDP Ops Portal**
Internal operations-facing UI. Allows WDP operations teams to manage disputes,
configure queues, manage users and organisations. Connects directly to API
Gateway — does not route through Akamai. Includes Disputes, Queues, User
Management, and Org Management sections. Dashboard section planned.

---

## Migration Progress Summary

| Status | Count | Components |
|--------|-------|------------|
| ✅ COMPLETE (individual file created and confirmed) | 1 | COMP-13 FileAcknowledgementProcessor |
| 📝 DRAFT (individual file created, architect confirmation pending) | 40 | COMP-01 through COMP-43 (excluding COMP-10, COMP-13, COMP-33, COMP-44) |
| 📋 PENDING (enterprise-owned, lower priority) | 1 | COMP-10 DM Mainframe |
| ⬜ NOT STARTED | 6 | COMP-33 (OrgManagementService — no repo found), COMP-44 (EDIAConsumer — planned), COMP-45, COMP-46, COMP-47 (File Generation — planned next sprint), COMP-48 (NYCEFileGenerationProcessor — planned) |
| 🔲 UI — separate action item | 2 | COMP-49 WDP Merchant Portal, COMP-50 WDP Ops Portal |

**Phase 1 and Phase 2 migration complete.** All 40 available component files created.
**Remaining:** COMP-33 (no repo), COMP-44/45/46/47/48 (planned features), COMP-49/50 (UI — separate action).

---

## Cross-Reference: Where Platform Sections Are Documented

| Section in WDP-COMPONENTS.md | Documented in |
|------------------------------|---------------|
| Part 3 — Kafka Event Bus (3.1, 3.2) | WDP-KAFKA.md |
| Part 6 — Acquiring Platform Integrations (6.1–6.5) | WDP-INTEGRATIONS.md |
| Part 7.3 — Disputes Section (UI) | WDP-COMP-49 and WDP-COMP-50 |
| Part 7.4 — Queues Section (UI) | WDP-COMP-50 (Ops Portal only) |
| Part 7.5 — User Management Section (UI) | WDP-COMP-49 and WDP-COMP-50 |
| Part 7.6 — Org Management Section (UI) | WDP-COMP-49 and WDP-COMP-50 |
| Part 7.7 — Dashboard Section (UI, Planned) | WDP-COMP-49 and WDP-COMP-50 |
| Appendix — Copilot CLI Question Sets | WDP-COMP-TEMPLATE.md (as usage guidance) |

---

## Adding New Components (Quarterly Update Protocol)

1. Assign the next sequential number (current last: 50)
2. Add a row to the registry table in the correct logical section
3. Write the one-line responsibility summary below the table
4. Create the component file using WDP-COMP-TEMPLATE.md
5. After the file is complete, update WDP-KAFKA.md and WDP-DB.md
   with entries from the new component
6. Update WDP-HANDOVER.md to note the new component and its number

---

*Last updated: April 2026*
*Current component count: 50*
*Next available number: 51*
