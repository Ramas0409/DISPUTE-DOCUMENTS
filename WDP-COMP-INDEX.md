# WDP-COMP-INDEX.md
**Worldpay Dispute Platform — Master Component Registry**
*Version: 2.1 | Reconciled: 2026-04-25*
*Source: v2.0 (April 2026) + 2026-04-18/23/24/25 source-verification reconciliation*

*v2.1 reconciliation scope: 20 of 40 DRAFT component files have undergone source-verified correction passes against their repositories. All carry "📝 DRAFT — source-verified [date], architect confirmation pending" status. Component descriptions updated where the audit changed key facts. Reconciled components: COMP-07, 08, 09, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 23, 24, 27, 37, 41, 43.*

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
- 📝 DRAFT 🔍 — DRAFT that has additionally undergone source-verified correction pass against the repository (date stamp shown per component below)
- 📋 PENDING — not yet migrated to individual file
- ⬜ NOT STARTED — no content exists yet (no repo or planned feature)
- 🔲 UI — tracked as separate action item

**v2.1 source-verification status:** 20 components carry the 🔍 marker. The corresponding source-verification dates are listed in the Reconciliation Tracker section below.

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
| 07 | VisaDisputeBatch | Batch/Scheduler | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-18) | WDP-COMP-07-VISA-DISPUTE-BATCH.md |
| 08 | FirstChargebackBatch | Batch/Scheduler | ✅ Production | 📝 DRAFT 🔍 v2.0 (2026-04-18) | WDP-COMP-08-FIRST-CHARGEBACK-BATCH.md |
| 09 | CaseFillingBatch | Batch/Scheduler | ✅ Production | 📝 DRAFT 🔍 v2.0 (2026-04-18) | WDP-COMP-09-CASE-FILLING-BATCH.md |

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
count must be 1 — replicas > 1 creates parallel polling risk (DEC-023, no
code-level guard).
⚠️ **(2026-04-18) Writer-ACK hazard:** mid-chunk JPA save failure does
not prevent ACK PUT to MCM; ACK exceptions are swallowed silently.
Items acknowledged off MCM with no corresponding outbox row are
unrecoverable. See WDP-NFRS.md RISK-052.

**09 — CaseFillingBatch**
Handles MasterCard dispute lifecycle stages after first chargeback: PAB,
ARB, PRA, AII, AIM. Polls MCM ReceiverCaseFiling queue via DataPower,
determines dispute stage from claim detail fields, encrypts PAN for new
claims, and writes to chbk_outbox_row. Stage determination uses priority-
ordered branch logic on caseType + rulingDate + issuerAction.
⚠️ **(2026-04-18) Writer swallows save exceptions** — chunk reports
success on DB save failure (RISK-053). **Skip paths write no audit row
at all** — null-validation, stage-determination, IDP, and encryption
failures leave zero database trace (RISK-063).

---

### Inbound Processing — File-Based Path

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 10 | DM Mainframe | External Infrastructure | ✅ Production | 📋 PENDING | WDP-COMP-10-DM-MAINFRAME.md |
| 11 | FileProcessor | Batch/Scheduler | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-18) | WDP-COMP-11-FILE-PROCESSOR.md |
| 12 | InboundDisputeEventScheduler | Batch/Scheduler + Kafka Producer | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-18) | WDP-COMP-12-INBOUND-EVENT-SCHEDULER.md |
| 13 | FileAcknowledgementProcessor | Batch/Scheduler | ✅ Production | 📝 DRAFT 🔍 v2.0 (2026-04-18) | WDP-COMP-13-FILE-ACK-PROCESSOR.md |

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
⚠️ **(2026-04-18) Silent file-loss findings:** DISCHYB and AMEXOPTB
qualifiers have no bean / no FileAcroEnum entry — files silently
discarded (RISK-054). DCPO evidence loop is empty-catch orphan generator
(RISK-064). DNWK has per-record silent row drop (RISK-065). DEC-004
non-numeric `acctNum` edge bypasses encryption (RISK-073). No K8s
probes, no HPA.

**12 — InboundDisputeEventScheduler**
Transactional outbox publisher. Runs five schedulers that read PENDING rows
from outbox tables and publish to the appropriate Kafka topics. Serves
as the Kafka publishing layer for VisaDisputeBatch, FirstChargebackBatch,
CaseFillingBatch, and FileProcessor — none of those write to Kafka directly.
⚠️ **(2026-04-18) Delivery model corrected:** **at-least-once with
duplicate-possible window** — mark-and-send within `@Transactional`,
broker ACK precedes TX commit (corrected from previously documented
at-most-once). Consumer-side `idempotency-key` dedup is the contracted
mitigation.
⚠️ **🔴 Replicas > 1 unmitigated:** no `@SchedulerLock`, no advisory
lock, no `SELECT FOR UPDATE`, no `synchronized` guard. Any production
replica value > 1 produces guaranteed duplicate Kafka publishes across
all five schedulers. Replica count XLD-templated; not visible in source.
RISK-038.

**13 — FileAcknowledgementProcessor**
Generates outbound acknowledgement files for inbound file jobs that required
confirmation. Polls file_job table for completed jobs with ack_required=true,
reads per-row detail from chbk_outbox_row, formats ACK files in four merchant-
specific formats (Walmart SigCap, Meijer, CapitalOne CMRTR, CapitalOne BJWC),
and uploads to S3. Documented in WDP-COMPONENTS.md sections 2.3.4 and 5.3.2
— these are the same deployable (wdp-evidence-ack-scheduler).
⚠️ **(2026-04-18) Three latent runtime bugs surfaced** during source-
verification, including a third headObject failure branch. RISK-056.
**Status reverted from COMPLETE to DRAFT 🔍** pending architect
re-confirmation.
⚠️ **`file_job` COMPLETED transition contract undocumented:** COMP-13
polls for `status IN (COMPLETED, ERROR)` but neither COMP-11 (writes
PROCESSING/PENDING) nor COMP-12 nor COMP-13 documents who writes the
COMPLETED transition. Candidate ADR-CAND-017.

---

### Core Processing — Event Consumers

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 14 | CaseCreationConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT 🔍 v2.0 (2026-04-18) | WDP-COMP-14-CASE-CREATION-CONSUMER.md |
| 15 | EvidenceConsumer | Kafka Consumer + Kafka Producer | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-18) | WDP-COMP-15-EVIDENCE-CONSUMER.md |
| 16 | BusinessRulesProcessor | Kafka Consumer + Kafka Producer | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-18) | WDP-COMP-16-BUSINESS-RULES-PROCESSOR.md |
| 17 | CaseExpiryUpdateConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-25) | WDP-COMP-17-CASE-EXPIRY-CONSUMER.md |
| 18 | NotificationOrchestrator | Kafka Consumer + Kafka Producer | ✅ Production | 📝 DRAFT 🔍 v2.0 (2026-04-18) | WDP-COMP-18-NOTIFICATION-ORCHESTRATOR.md |

**14 — CaseCreationConsumer**
Primary case creation component for all non-NAP dispute events. Consumes
from new-case-events topic. Enriches events with merchant and transaction
data from the relevant acquiring platform API (CORE via MerchantTransactionService,
LATAM and VAP via direct API calls; PIN shares the CORE URL). Creates dispute
cases in WDP Core via CaseManagementService. Handles transient PAN decryption
for enrichment calls — clear PAN never persisted. Pre-ACK (DEC-005).
⚠️ **(2026-04-18) Confirmed NOT a publisher of `business-rules`** — closes
prior open question; no `KafkaTemplate`, no `ProducerFactory`, no topic
reference in source. Confirms 6 (not 7) publishers on `business-rules`.
⚠️ **Zero `@Transactional` annotations** anywhere on the processing path —
all DB writes auto-commit. Empty `CommonErrorHandler{}` silently swallows
deserialisation/application exceptions. RISK-025.

**15 — EvidenceConsumer**
Consumes from case-evidence-events topic and attaches evidence documents to
existing dispute cases. Two paths: WDP path (PIN always; CORE when
coreMigrationFlag=true) uploads via DocumentManagementService and publishes
to business-rules; V3 legacy path (CORE when flag=false) queries IBM DB2 and
uploads to V3 Core endpoint. Pre-ACK (DEC-005). Synchronous publish inside
`@Transactional` via `kafkaTemplate.send(...).get()` — kafka-before-commit
pattern (DEC-001 deviation).
⚠️ **(2026-04-18) V3 silent-ATTACHED defect (RISK-029):** MISCDOC, DRFTDOC,
RESPQDOC, ISSRQDOC document types mark `attachment_status=ATTACHED` with
no upload (4 doc types).
⚠️ **(2026-04-18) V3 PATCH ghost-upload (RISK-030):** failed
ownership-transfer leaves V3 Core with the document and WDP DB unmarked.

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
predecessor control (channel_type=EXPIRY_EVENTS). Terminal consumer — no
Kafka publish.
⚠️ **(2026-04-25) Cross-action predecessor interference (RISK-045):** query
not scoped on `i_action_seq` — stuck row for action A blocks events for
action B on same case.
⚠️ **🔴 Non-atomic outbox + case_expiry split (RISK-031):** outbox INSERT
and `case_expiry` write run in separate transactional boundaries. ACK
timing is inconsistent (Path A pre-ACK, Path B mid-flow ACK). Both
unrecoverable on every successful execution path.

**18 — NotificationOrchestrator** *(repository: wp-mfd/wdp-outgoing-consumer)*
Central outbound routing component. Consumes from outgoing-events topic and
fans out to up to four simultaneous outputs via four independent filter methods.
Publishes to case-action-events, core-request-events, and external-request-events.
Also writes to wdp.file_generation_event to stage requests for file-based output
components. Uses wdp.bre_orchestration_outbox for idempotency and retry (component=
NOTIFICATION_ORCHESTRATOR rows). ACK committed after outbox INSERT but before
Kafka publishes (DEC-005 deviation). Does NOT publish to internal-integration-events.
⚠️ **(2026-04-18) DEC-001 PARTIAL** — `wdp.bre_orchestration_outbox` is used
as the outbox, but **zero `@Transactional` annotations** anywhere. Four
distinct write points (3a ERROR, 3d PUBLISHED, 6 ERROR/PENDING_DEFERRED,
7e SUCCESS/FAILED) are independent auto-commits. PUBLISHED orphan rows have
no automatic re-drive (Scheduler4 reads only FAILED/PENDING_DEFERRED).
⚠️ Confirmed NOT a writer of `wdp.outgoing_event_outbox` (corrected from
v2.0 — zero references in source).

---

### Core Processing — Case Action Services

| # | Component | Type | Prod Status | Doc Status | File |
|---|-----------|------|-------------|------------|------|
| 19 | AcceptService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT 🔍 v2.0 (2026-04-23) | WDP-COMP-19-ACCEPT-SERVICE.md |
| 20 | ContestService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT 🔍 v2.0 (2026-04-23) | WDP-COMP-20-CONTEST-SERVICE.md |
| 21 | ChargebackService | REST API | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-25) | WDP-COMP-21-CHARGEBACK-SERVICE.md |
| 22 | DisputeService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-22-DISPUTE-SERVICE.md |
| 23 | CaseManagementService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-23) | WDP-COMP-23-CASE-MANAGEMENT-SERVICE.md |
| 24 | CaseActionService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-23) | WDP-COMP-24-CASE-ACTION-SERVICE.md |
| 25 | NotesService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT | WDP-COMP-25-NOTES-SERVICE.md |
| 26 | QuestionnaireService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-26-QUESTIONNAIRE-SERVICE.md |

**19 — AcceptService**
Processes merchant decisions to accept a dispute. Calls Visa and MasterCard
APIs directly and synchronously. Publishes AcceptEvent to internal-integration-events
(caseNumber key — DEC-003 deviation). For NAP disputes, NAPOutcomeProcessor
delivers the outcome to NAP-DPS. No Kafka DLQ or outbox — DEC-001 deviation.
⚠️ **🔴 (2026-04-23) NAP split-brain on MC CHI / AMEX / DISCOVER (RISK-028):**
silent no-op for these branches still triggers AcceptEvent publish — NAP-DPS
notified while card network was never asked. Same severity class as
DEC-019/020. ADR-CAND-001 pending architect decision.
⚠️ **Compensation has no inner try/catch (RISK-044)** — secondary failure
masks original business exception. Likely shared with COMP-20.

**20 — ContestService**
Processes merchant decisions to contest a dispute. Calls Visa and MasterCard
APIs directly and synchronously. Publishes ContestEvent to internal-integration-events
(merchantId key — DEC-003 compliant). Up to 2 publishes per request when CRMR
action code present. No Kafka DLQ or outbox — DEC-001 deviation.
⚠️ **(2026-04-23) MCM stage-dependent path:** CH1/RE2 →
`createSecondPresentmentChargeback` (POST); PAB → `retrieveClaim +
updateCase(action=REJECT)` (GET then PUT); CHI silent no-op on both NAP
and PIN.
⚠️ **Compensation no inner try/catch (RISK-044)** — confirmed pattern
shared with COMP-19.

**21 — ChargebackService**
**Sole externally-exposed WDP REST API** accessible to merchants and partners
via APIGEE → Akamai. Handles both dispute read operations (case search, detail,
documents) and action operations (contest, accept, add note, change owner,
document upload). Platform routing derived entirely from inbound caseId format.
Two runtime authorization modes toggled by entitlement_flag (legacy ACL vs
newer Cert eAPI Entitlement Lookup — currently inactive in prod).
⚠️ **(2026-04-25) Largest outbound-integration owner:** 38 distinct
downstream call sites across 12 target applications, all on a single shared
`RestTemplate` with no pool, no timeouts, no retries, no circuit breaker.
⚠️ **JustAI is active in COMP-21 (corrected 2026-04-25):** partner identity
via JWT consumer name `JUSTTAI` (double-T spelling in source) — contradicts
v1.0 platform-wide claim of "planned only" (which is correct only for COMP-41
outbound).
⚠️ Parallel ACL+lookup pattern is **effectively serial under production
sizing** (asyncExecutor core=1/max=1/queue=5). RISK-026.
⚠️ LATAM silent fall-through (RISK-027). Surfaces full `cardNumber` in two
response models (RISK-051).

**22 — DisputeService**
Read-and-orchestration layer only — owns no database state, performs no writes.
Two independent endpoints: POST /summary (dispute summary aggregation from WDP
Aurora or IBM DB2 depending on core_migration_status flag) and POST
/{platform}/cases/{caseNumber}/documents (internal-firm document upload
orchestration). **Kafka-free at runtime** — Kafka producer to `business-rules`
is wired in source but all publish call sites are commented out in production.
RISK-021.

**23 — CaseManagementService**
Authoritative owner of the dispute case record across all five platforms (NAP,
PIN, CORE, VAP, LATAM). Single write target for case creation and updates.
Owns separate nap and wdp PostgreSQL schemas — never cross-schema in one
transaction. Publishes BusinessRuleEvent to business-rules Kafka topic
(caseNumber key — DEC-003 deviation) — **kafka-before-commit pattern**:
publish inside `@Transactional` BEFORE commit (corrected from v1.0's "after
commit").
⚠️ **(2026-04-23) Owns 6 tables, not 7** — `wdp.dispute_event_change_log`
removed (table does not exist in source).
⚠️ **DEC-019 PostgreSQL violation:** clear PAN written to
`nap.case.I_ACCT_CDH` *(corrected from typo `I_ACCI_CDH`)* and
`wdp.CASE.I_ACCT_CDH` on standard case creation — encryption only occurs
during transaction enrichment flow (PIN/CORE only). RISK-004.
⚠️ **(2026-04-23) US create path duplicate-NOTES-insert defect (RISK-032):**
two identical `USNotesEntity` save blocks execute when `notesRequest != null`
— two rows per create.
⚠️ **(2026-04-23) Cross-component blind-merge (RISK-033):** writes into
`NAP.DISPUTE_EVENT_CONSUMER_ERROR` (COMP-05-owned) with no owner check.

**24 — CaseActionService** *(repository: `gcp-cas-actions-service`, artifact `case-actions-service`)*
Handles all case action lifecycle operations across NAP and US platforms —
creating, updating, and closing actions; ownership transfers; case open/close
transitions. Writes wdp.case, wdp.action, nap.case, nap.action, and
conditionally wdp.chbk_outbox_row and wdp.notes in the same transaction.
Publishes to business-rules topic (BRE inside `@Transactional` — DEC-001
PARTIAL) and a second ActionEvent topic on EP 2/8/9 when napUpdateEvent=true.
⚠️ **No RBAC enforcement (DEC-018, RISK-005)** — `RestInvoker.authorizeUser()`
defined but never called.
⚠️ **(2026-04-23) ActionEvent post-commit split-brain on EP 2/8/9
(RISK-046):** domain commit succeeds, ActionEvent publish lost on broker
failure. Genuine post-commit split-brain (BRE publish is NOT — corrected
from v1.0).
⚠️ **EP-9 three-transaction sequence with no compensation (RISK-039).**
NAP/US asymmetry on EP 5 — `chbkOutbox` ignored on NAP (RISK-047).
Open-action constraint enforced in-memory only (RISK-048).

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
| 27 | CaseSearchService | REST API | ✅ Production | 📝 DRAFT 🔍 v2.0 (2026-04-24) | WDP-COMP-27-CASE-SEARCH-SERVICE.md |
| 28 | DisplayCodeService | REST API | ✅ Production | 📝 DRAFT | WDP-COMP-28-DISPLAY-CODE-SERVICE.md |

**27 — CaseSearchService**
Platform-wide read layer for dispute case data across all five acquiring
platforms. 14 active REST endpoints across six controllers. Platform routing
via {platform}/{region} path variable selecting between nap.* and wdp.* schemas.
Case detail endpoints use parallel async fan-out (Spring @Async, 10-thread pool)
calling up to five downstream services concurrently. Display code lookups
in-memory cached for JVM lifetime.
⚠️ **🔴 (2026-04-24) Concurrent-locker race on case lock UPDATE (RISK-041):**
no `@Transactional`, no `SELECT FOR UPDATE`, no optimistic version. Two
concurrent lockers can both UPDATE; second silently overwrites first.
⚠️ **🔴 `/lft` authorization derives from request body (RISK-042):**
`LftSearchParams.isInternal` from POST body, not from JWT. Security gap.
⚠️ **🔴 Queue-criterion `value` SQL concatenation (RISK-043):** latent
SQL-injection surface — gated on whether production queue-criterion
content can contain single quotes.
⚠️ External VAP/LATAM auth fall-through (RISK-078) — no entity-scope
authorization service called for these platforms.

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
| 37 | DocumentManagementService | REST API + Kafka Producer | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-23) | WDP-COMP-37-DOCUMENT-MANAGEMENT-SERVICE.md |
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
(NAP document attachments), ContestService (contest attachments), ChargebackService
(merchant uploads), and merchant UI submissions. Only WDP component using
**S3 + DynamoDB + dual-PostgreSQL datasources** as primary storage.
⚠️ **(2026-04-23) 6th publisher of `business-rules` Kafka topic** —
caseNumber key (DEC-003 deviation, uniform with other 5 publishers).
Endpoints 1, 9, 10 (uploads) and Endpoint 11 (questionnaire) publish;
Endpoint 8 NAP base64 path never publishes. Endpoint 11 wraps publish in
`@Transactional(rollbackOn=Exception.class)` — stronger atomicity than
other publishers (candidate stopgap pattern, ADR-CAND-009).
⚠️ **🔴 5-step non-atomic write chain (RISK-034):** S3 → DDB → desk →
Kafka → action-indicator. No compensation at any boundary; failure after
Kafka is unrecoverable without manual BR re-drive.
⚠️ DynamoDB duplicate-check race (RISK-061). RestTemplate has 13+ call
sites + a second inline `new RestTemplate()` in `RestInvoker.postData()`,
none with timeouts.

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
| 41 | ThirdPartyNotificationConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT 🔍 v1.1 (2026-04-25) | WDP-COMP-41-THIRD-PARTY-NOTIFICATION-CONSUMER.md |
| 42 | BENConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT | WDP-COMP-42-BEN-CONSUMER.md |
| 43 | CoreNotificationConsumer | Kafka Consumer | ✅ Production | 📝 DRAFT 🔍 v2.0 (2026-04-25) | WDP-COMP-43-CORE-NOTIFICATION-CONSUMER.md |
| 44 | EDIAConsumer | Kafka Consumer + Kafka Producer | 🔴 Planned | ⬜ NOT STARTED | WDP-COMP-44-EDIA-CONSUMER.md |

**41 — ThirdPartyNotificationConsumer**
Consumes external-request-events and delivers dispute event notifications to
Signifyd via REST API. Applies own filtering to determine which events require
third-party notification. After receiving notification Signifyd calls back
ChargebackService for dispute details. Idempotency via wdp.outgoing_event_outbox
(channel_type=GP_EVENTS — *corrected 2026-04-25 from `GF_EVENTS`*).
⚠️ **(2026-04-25) JustAI scope clarification:** **JustAI is planned only
for outbound notification in this component** — no JustAI reference exists
in COMP-41 codebase. Signifyd is the sole live vendor in COMP-41 outbound.
**JustAI IS active in COMP-21 inbound partner identification.**
⚠️ **🔴 Three distinct PUBLISHED-orphan paths (RISK-040, extends RISK-015):**
(a) post-ACK crash, (b) Signifyd "NO_DATA_FROM_SIGNIFYD" empty body,
(c) final outbox UPDATE failure. All invisible to Scheduler3 if it reads
only FAILED/PENDING_DEFERRED. OQ-COMP41-1.
⚠️ Silent `@Cacheable` no-op (RISK-055) — `@EnableCaching` absent.
Dead Spring Retry imports (RISK-080).

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
and retry. **Sole WDP writer to IBM DB2 BC schema.**
⚠️ **(2026-04-25) Read-list corrected:** DB2 reads are limited to
`BC.TBC_DM_CASE` only via `findByCaseId` and `findByWdpCaseNumber`.
`BC.TBC_DM_OCCUR` and `BC.TBC_DM_NOTES` are write-only. Enrichment is
fully REST-driven.
⚠️ **🔴 DEC-019 DB2 sibling (RISK-035):** clear PAN written to
`BC.TBC_DM_CASE.I_ACCT_CDH` *(corrected from `I_ACCT_CDR` in v2.0)* on
Step 7 CREATE + actionSeq=01 path. Architect decision required —
extend DEC-019 or remediate (ADR-CAND-004).
⚠️ **🔴 Silent-loss window (RISK-036):** between ACK and FAILED-write,
PostgreSQL unavailability defeats fallback.
⚠️ **🔴 REST PUT inside DB2 `@Transactional` on CREATE (RISK-037)** — REST
latency holds DB2 locks. No K8s probes (RISK-024).

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

## v2.1 Reconciliation Tracker

This table records the source-verification dates for the 20 components reconciled between 2026-04-18 and 2026-04-25. Each component has been source-verified against its production repository by Claude Code; architect confirmation is the next gate.

| Date | Component | Version | Repo |
|------|-----------|---------|------|
| 2026-04-18 | COMP-07 VisaDisputeBatch | v1.0 → v1.1 | (per source) |
| 2026-04-18 | COMP-08 FirstChargebackBatch | v1.0 → v2.0 | (per source) |
| 2026-04-18 | COMP-09 CaseFillingBatch | v1.0 → v2.0 | (per source) |
| 2026-04-18 | COMP-11 FileProcessor | v1.0 → v1.1 | (per source) |
| 2026-04-18 | COMP-12 InboundDisputeEventScheduler | v1.0 → v1.1 | wdp-chargeback-evidence-event-scheduler |
| 2026-04-18 | COMP-13 FileAcknowledgementProcessor | v1.0 → v2.0 | (per source) |
| 2026-04-18 | COMP-14 CaseCreationConsumer | v1.0 → v2.0 | gcp-case-creation-consumer |
| 2026-04-18 | COMP-15 EvidenceConsumer | v1.0 → v1.1 | (per source) |
| 2026-04-18 | COMP-16 BusinessRulesProcessor | v1.0 → v1.1 | wdp-business-rules-processor |
| 2026-04-18 | COMP-18 NotificationOrchestrator | v1.0 → v2.0 | wp-mfd/wdp-outgoing-consumer |
| 2026-04-23 | COMP-19 AcceptService | v1.0 → v2.0 | mdva-gcp-disputes-accept-service |
| 2026-04-23 | COMP-20 ContestService | v1.0 → v2.0 | (per source) |
| 2026-04-23 | COMP-23 CaseManagementService | v1.0 → v1.1 | mdws-gcp-case-management-service |
| 2026-04-23 | COMP-24 CaseActionService | v1.0 → v1.1 | gcp-cas-actions-service (artifact: case-actions-service) |
| 2026-04-23 | COMP-37 DocumentManagementService | v1.0 → v1.1 | gcp-document-management-service |
| 2026-04-24 | COMP-27 CaseSearchService | v1.0 → v2.0 | (per source) |
| 2026-04-25 | COMP-17 CaseExpiryUpdateConsumer | v1.0 → v1.1 | (per source) |
| 2026-04-25 | COMP-21 ChargebackService | v1.0 → v1.1 | mdvs-gcp-chargeback-service |
| 2026-04-25 | COMP-41 ThirdPartyNotificationConsumer | v1.0 → v1.1 | (per source) |
| 2026-04-25 | COMP-43 CoreNotificationConsumer | v1.0 → v2.0 | (per source) |

All 20 entries have a corresponding Reconciled entry in WDP-CHANGE-LOG.md.

---

## Migration Progress Summary

| Status | Count | Components |
|--------|-------|------------|
| ✅ COMPLETE (individual file created and confirmed) | 0 | (COMP-13 reverted to DRAFT after 2026-04-18 source-verification surfaced 3 latent runtime bugs) |
| 📝 DRAFT 🔍 (source-verified, architect confirmation pending) | 20 | COMP-07, 08, 09, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 23, 24, 27, 37, 41, 43 |
| 📝 DRAFT (Copilot CLI analysis, source-verification pending) | 21 | COMP-01, 02, 03, 04, 05, 06, 22, 25, 26, 28, 29, 30, 31, 32, 34, 35, 36, 38, 39, 40, 42 |
| 📋 PENDING (enterprise-owned, lower priority) | 1 | COMP-10 DM Mainframe |
| ⬜ NOT STARTED | 6 | COMP-33 (OrgManagementService — no repo found), COMP-44 (EDIAConsumer — planned), COMP-45, COMP-46, COMP-47 (File Generation — planned next sprint), COMP-48 (NYCEFileGenerationProcessor — planned) |
| 🔲 UI — separate action item | 2 | COMP-49 WDP Merchant Portal, COMP-50 WDP Ops Portal |

**Phase 1, 2, and 3 (source-verification) progress:**
- All 41 available component files have been created.
- 20 of 41 have undergone source-verified correction passes against their repositories between 2026-04-18 and 2026-04-25.
- Remaining 21 source-verifications are pending (lower-priority components — e.g. COMP-04, 05, 06 are decommission-scoped).
- No component has yet been architect-confirmed. The "✅ COMPLETE" criterion (architect-confirmed) is the next gate.

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

*Last updated: 2026-04-25 (v2.1 reconciliation)*
*Current component count: 50*
*Next available number: 51*
